# Building Agents with LangGraph

The agentic layer of the Incident Response Framework is a **LangGraph `StateGraph`** containing a Supervisor and six specialist agents. Each agent is a pure function that reads a shared `AgentState`, performs its work (tool calls, LLM inference, DB writes), and returns a partial state update. Routing between agents is handled by the Supervisor via a conditional edge, which keeps the control flow fast and deterministic while still giving every agent full access to the running state.

This document is a practical guide to designing, implementing, and orchestrating those agents.

---

## Table of Contents

1. [LangGraph in One Page](#1-langgraph-in-one-page)
2. [Shared AgentState](#2-shared-agentstate)
3. [Supervisor / Orchestrator](#3-supervisor--orchestrator)
4. [Incident Detector](#4-incident-detector)
5. [Root Cause Finder](#5-root-cause-finder)
6. [Incident Mitigator and the Feedback Loop](#6-incident-mitigator-and-the-feedback-loop)
7. [Incident Communicator](#7-incident-communicator)
8. [Incident Summarizer](#8-incident-summarizer)
9. [Assembling the StateGraph](#9-assembling-the-stategraph)
10. [MySQL Checkpointing](#10-mysql-checkpointing)
11. [Triggering the Graph](#11-triggering-the-graph)
12. [Testing Agents](#12-testing-agents)

---

## 1. LangGraph in One Page

LangGraph is a library for building stateful, multi-agent workflows on top of LangChain. The core concepts:

- **StateGraph** — a directed graph of nodes that mutate a shared state dict.
- **Nodes** — plain functions (sync or async) that take the current state and return a partial state update.
- **Edges** — either fixed or *conditional*. A conditional edge is a function that inspects the state and returns the name of the next node.
- **Reducers** — declared on state fields to merge updates (for example, `add_messages` appends rather than overwrites message lists).
- **Checkpointing** — every node boundary can persist the state to a backend. LangGraph ships with an in-memory and SQLite saver out of the box, and custom backends (like the MySQLSaver below) plug in via `BaseCheckpointSaver`.
- **Feedback loops** — implemented as conditional edges that route control back to an earlier node based on state fields.

## 2. Shared AgentState

Every agent reads and writes the same `AgentState` TypedDict. Keeping the state explicit makes reasoning, testing, and checkpointing straightforward.

```python
# agents/state.py
from typing import TypedDict, Optional, List, Annotated
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # Workflow control
    phase: str                         # current pipeline phase
    session_id: str                    # MySQL session UUID
    feedback_iteration: int            # feedback loop counter

    # Incident data
    incident_id: Optional[str]
    severity: Optional[str]
    anomaly_nodes: List[dict]          # from Incident Detector
    root_causes: List[dict]            # from Root Cause Finder
    mitigation_workflows: List[dict]   # from Incident Mitigator
    mitigation_confidence: float       # drives feedback loop
    communications_sent: List[dict]
    incident_summary: Optional[str]

    # LangGraph message history
    messages: Annotated[list, add_messages]

    # Feedback
    feedback_request: Optional[str]    # mitigator → root_cause_finder
```

A few conventions worth noting:

- `phase` is the **single source of truth for routing**. The Supervisor never calls an LLM; it just reads `phase`.
- Lists are returned as full replacements by each agent, while `messages` uses the `add_messages` reducer so LangChain message history accumulates naturally.
- `feedback_iteration` lets the mitigator cap the number of loop cycles it can trigger before escalating.

## 3. Supervisor / Orchestrator

The Supervisor is a **conditional router** — no LLM, no tools, just a deterministic function that reads the current phase from state and returns the name of the next node:

```python
# agents/supervisor.py
from langgraph.graph import END
from .state import AgentState

def supervisor(state: AgentState) -> str:
    phase = state["phase"]
    if phase == "detect":      return "incident_detector"
    if phase == "root_cause":  return "root_cause_finder"
    if phase == "mitigate":    return "incident_mitigator"
    if phase == "feedback":    return "root_cause_finder"
    if phase == "communicate": return "incident_communicator"
    if phase == "summarize":   return "incident_summarizer"
    return END
```

Because it doesn't call an LLM, routing adds no latency and is trivially testable.

## 4. Incident Detector

**Goal** — decide whether currently active anomalies in the graph constitute an actionable incident.

**Inputs (from state):** `phase = 'detect'` — no prior data needed.

**Tools used:** `list_anomalies`, `root_cause_analysis` (both via the Graph DB MCP).

**Outputs (to state):**

| Field | Meaning |
|-------|---------|
| `phase` | `'root_cause'` if an incident is confirmed, `END` otherwise |
| `anomaly_nodes` | List of dicts describing affected topology nodes |
| `incident_id` | Newly created `IncidentTicket.id` in graphserv |
| `severity` | Derived severity label |

**Design pattern:**

```python
async def incident_detector(state: AgentState) -> dict:
    llm = get_llm("incident_detector").bind_tools(graph_tools)
    anomalies = await call_tool("list_anomalies", status="open", limit=20)

    decision = await llm.ainvoke([
        SystemMessage("Decide whether these anomalies form an incident. Return JSON: {incident: bool, severity: str, affected_nodes: [..]}"),
        HumanMessage(anomalies),
    ])
    data = json.loads(decision.content)

    if not data["incident"]:
        return {"phase": END, "anomaly_nodes": []}

    incident_id = await call_tool("create_incident_ticket", severity=data["severity"], status="active")
    return {
        "phase": "root_cause",
        "incident_id": incident_id,
        "severity": data["severity"],
        "anomaly_nodes": data["affected_nodes"],
    }
```

The LLM's `format="json"` setting (from the LLM factory) ensures the response is valid JSON that can be parsed without cleanup.

## 5. Root Cause Finder

**Goal** — produce a coherent root cause hypothesis with evidence.

**Inputs:** `anomaly_nodes`, optionally `feedback_request` from the mitigator.

**Tools used:**

| Tool | Purpose |
|------|---------|
| `root_cause_analysis` (Graph DB MCP) | Graph traversal — finds upstream origins |
| `get_change_tickets` (Graph DB MCP) | Recent changes that may have caused the anomaly |
| Qdrant `rca_documents` search | Past RCAs for the same node or anomaly type |
| Qdrant `change_context` search | Semantic search for relevant recent changes |

**Output:** `root_causes` — a list of dicts, each containing `hypothesis`, `evidence`, `confidence`, and the affected graph nodes.

The agent's prompt asks the LLM to weigh graph-traversal results and RAG hits together and to produce a short, citable explanation. When the node is entered via the feedback loop (`feedback_request` is set), the agent is explicitly instructed to widen the search — deeper `maxDepth`, additional collections, or fresh queries based on the feedback message.

## 6. Incident Mitigator and the Feedback Loop

**Goal** — select and execute the right mitigation workflow, or ask the Root Cause Finder for a deeper look if confidence is too low.

**Inputs:** `root_causes`, `feedback_iteration`.

**Tools used:** `search_mitigation_workflows`, `execute_mitigation_step` (both via the Mitigation MCP), `update_node_status` (Graph DB MCP).

The mitigator searches `mitigation_workflows` using the root cause summary as the query. If the top result's confidence exceeds the threshold (default `0.75`), the mitigator confirms the workflow and calls `execute_mitigation_step` for each step in the chosen runbook. If the confidence is below the threshold, the mitigator sets `feedback_request` and routes back to the Root Cause Finder:

```python
# Feedback loop trigger inside the mitigator
if top_workflow.score < CONFIDENCE_THRESHOLD and state["feedback_iteration"] < MAX_ITERATIONS:
    return {
        "phase": "feedback",
        "feedback_request": (
            f"Need deeper analysis of {root_cause_area}. "
            f"Current top workflow score only {top_workflow.score:.2f}."
        ),
        "feedback_iteration": state["feedback_iteration"] + 1,
    }
```

A row is inserted into `feedback_log` every time the loop triggers, making it easy to trace why the system decided to keep looking. The loop terminates either when confidence clears the threshold or when `feedback_iteration` reaches `MAX_ITERATIONS`, at which point the agent escalates the incident instead of executing automatically.

## 7. Incident Communicator

**Goal** — draft and dispatch channel-appropriate messages about the incident.

**Inputs:** `root_causes`, `mitigation_workflows`, `severity`.

**Tools used:** all six tools on the Comms MCP (`send_email`, `send_slack`, `send_teams`, `send_sms`, `page_oncall`, `update_ticket_comms`).

The LLM (Mistral 7B by default) receives a prompt that contains the structured incident summary and a list of target channels. It is asked to emit terse messages for PagerDuty and SMS and more detailed messages for Email and Teams. The agent then calls the corresponding Comms MCP tools and records what was sent in `communications_sent`, before updating the incident ticket with communication timestamps.

## 8. Incident Summarizer

**Goal** — synthesise the full incident story for memory and recall.

**Inputs:** the full state.

**Outputs:**

- `incident_summary` — a concise human-readable story (timeline, root cause, mitigation, outcome).
- A new point in Qdrant's `incident_summaries` collection, embedded and tagged for future RAG.
- A `sessions` row update in MySQL with the final `status`, `summary`, and timestamps.
- New entries in `agent_memory` for any patterns the Summarizer extracted (for example, "latency spikes on `app-123` after VPC config changes").

## 9. Assembling the StateGraph

All nodes and edges live in `graph/workflow.py`:

```python
# graph/workflow.py
from langgraph.graph import StateGraph, START, END
from .state import AgentState
from .agents import (
    supervisor,
    incident_detector, root_cause_finder, incident_mitigator,
    incident_communicator, incident_summarizer,
)

builder = StateGraph(AgentState)

# Register every worker node
builder.add_node("incident_detector",    incident_detector)
builder.add_node("root_cause_finder",    root_cause_finder)
builder.add_node("incident_mitigator",   incident_mitigator)
builder.add_node("incident_communicator",incident_communicator)
builder.add_node("incident_summarizer",  incident_summarizer)

# Supervisor is a conditional edge, not a node
builder.set_entry_point("incident_detector")

builder.add_conditional_edges("incident_detector",     supervisor)
builder.add_conditional_edges("root_cause_finder",     supervisor)
builder.add_conditional_edges("incident_mitigator",    supervisor)   # may route to root_cause_finder
builder.add_conditional_edges("incident_communicator", supervisor)
builder.add_conditional_edges("incident_summarizer",   supervisor)

graph = builder.compile(checkpointer=mysql_saver)
```

Because the Supervisor is a pure function of `phase`, any node can advance the pipeline simply by setting the next phase on its returned state dict. The mitigator's "send me back" loop is just another value of `phase` — no special wiring required.

## 10. MySQL Checkpointing

LangGraph ships with a SQLite saver out of the box. For this project a thin MySQL adapter is implemented against `BaseCheckpointSaver` so that every state transition is persisted into the `checkpoints` table. This gives:

- **Resumable workflows** — kill the agent container mid-run and restart from the last checkpoint.
- **Replay** — step through a past incident for debugging.
- **Human-in-the-loop** — pause at a chosen node and require explicit approval before continuing.

Reference skeleton:

```python
# memory/mysql_saver.py
from langgraph.checkpoint.base import BaseCheckpointSaver
import aiomysql, pickle

class MySQLSaver(BaseCheckpointSaver):
    def __init__(self, pool: aiomysql.Pool):
        self.pool = pool

    async def aput(self, config, checkpoint, metadata, new_versions):
        async with self.pool.acquire() as conn, conn.cursor() as cur:
            await cur.execute(
                """
                INSERT INTO checkpoints
                    (thread_id, checkpoint_ns, checkpoint_id, parent_checkpoint_id, type, checkpoint, metadata)
                VALUES (%s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE checkpoint = VALUES(checkpoint), metadata = VALUES(metadata)
                """,
                (
                    config["configurable"]["thread_id"],
                    config["configurable"].get("checkpoint_ns", ""),
                    checkpoint["id"],
                    checkpoint.get("parent_id"),
                    checkpoint.get("type"),
                    pickle.dumps(checkpoint),
                    pickle.dumps(metadata),
                ),
            )

    async def aget_tuple(self, config):
        # Restore latest checkpoint for the given thread
        ...
```

See the [Database Setup](./02-database-setup.md#3-mysql) document for the `checkpoints` table DDL.

## 11. Triggering the Graph

A simple CLI (`trigger.py`) runs the graph either on demand or on a polling loop:

```bash
python trigger.py --poll                   # polls graphserv every POLL_INTERVAL_SECONDS
python trigger.py --incident INC-101       # run once for a specific incident
```

The polling loop calls `list_anomalies` on graphserv every 60 seconds (configurable via `POLL_INTERVAL_SECONDS`). When new open anomalies are detected, it starts a fresh thread in LangGraph with a new `session_id` and routes straight into the Incident Detector.

## 12. Testing Agents

### 12.1 Unit Tests

- Use `FakeListChatModel` to stub LLM responses.
- Use pytest `monkeypatch` to replace MCP tool callables with canned returns.
- Drive each agent with a handcrafted `AgentState` and assert the returned partial state.

| Test Target | File | Assertions |
|-------------|------|------------|
| Incident Detector | `tests/test_detector.py` | `incident=False` on empty anomalies; `incident=True` with severity on populated anomalies |
| Root Cause Finder | `tests/test_rcf.py` | Merges graph hits and Qdrant hits into a single hypothesis |
| Incident Mitigator | `tests/test_mitigator.py` | Low confidence triggers feedback; high confidence returns workflow steps |
| Feedback Loop | `tests/test_feedback.py` | Loop terminates at `MAX_ITERATIONS`; one `feedback_log` row per cycle |
| MySQL Saver | `tests/test_mysql_saver.py` | `aput` stores; `aget_tuple` restores identical state |

### 12.2 Integration Tests

With the full Docker Compose stack running (`docker compose --profile test up`):

- `tests/integration/test_full_pipeline.py` — end-to-end happy path. Seed Neo4j with a known topology, insert a test anomaly, assert that the `sessions` row and the `incident_summaries` Qdrant point are written correctly.
- `tests/integration/test_feedback_loop.py` — force a low-confidence Qdrant result and verify the loop triggers and eventually resolves.
- `tests/integration/test_resume.py` — kill the agent container mid-run, restart, and verify LangGraph resumes from the last MySQL checkpoint.

### 12.3 Observability

Every agent emits structured JSON log lines. During development a simple tail gives a full view of a run:

```bash
docker compose logs -f agents | python -m json.tool
```

Enable LangSmith tracing when you need visual traces of tool calls, LLM prompts, and state transitions:

```bash
LANGCHAIN_TRACING_V2=true LANGCHAIN_API_KEY=<key> python trigger.py --poll
```

---

*See also: [Incident Response Framework](./01-incident-response-framework.md) · [MCP Servers](./04-mcp-servers.md) · [Database Setup](./02-database-setup.md)*
