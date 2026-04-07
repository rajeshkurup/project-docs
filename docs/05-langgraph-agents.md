# Building Agents with LangGraph

The agentic layer of the Incident Response Framework is a **LangGraph `StateGraph`** containing a Supervisor and five specialist agents. Each agent is a pure async function that reads a shared `AgentState`, performs its work (tool calls, LLM inference, Qdrant reads/writes), and returns a partial state update. A sixth cross-cutting agent — the **Incident Communicator** — is invoked by the Supervisor after every significant pipeline event to print a formatted banner and save a rolling summary to Qdrant.

This document is a practical guide to the design, implementation, and orchestration of those agents as they are actually built and running today.

---

## Table of Contents

1. [LangGraph in One Page](#1-langgraph-in-one-page)
2. [Shared AgentState](#2-shared-agentstate)
3. [Supervisor / Orchestrator](#3-supervisor--orchestrator)
4. [Agent Pipeline Overview](#4-agent-pipeline-overview)
5. [Incident Detector](#5-incident-detector)
6. [Incident Communicator](#6-incident-communicator)
7. [Root Cause Finder](#7-root-cause-finder)
8. [Incident Mitigator and the Feedback Loop](#8-incident-mitigator-and-the-feedback-loop)
9. [Incident Summarizer](#9-incident-summarizer)
10. [Qdrant Vector Collections](#10-qdrant-vector-collections)
11. [Assembling the StateGraph](#11-assembling-the-stategraph)
12. [MySQL Checkpointing](#12-mysql-checkpointing)
13. [Triggering the Graph](#13-triggering-the-graph)
14. [Testing Agents](#14-testing-agents)

---

## 1. LangGraph in One Page

LangGraph is a library for building stateful, multi-agent workflows on top of LangChain. The core concepts:

- **StateGraph** — a directed graph of nodes that mutate a shared state dict.
- **Nodes** — plain async functions that take the current state and return a partial state update.
- **Edges** — either fixed or *conditional*. A conditional edge is a function that inspects state and returns the name of the next node.
- **Reducers** — declared on state fields to merge updates (e.g. `add_messages` appends rather than overwrites message lists).
- **Checkpointing** — every node boundary can persist state to a backend. The custom `MySQLSaver` plugs in via `BaseCheckpointSaver`.
- **Feedback loops** — conditional edges routing control back to an earlier node based on a state field value.

---

## 2. Shared AgentState

Every agent reads and writes the same `AgentState` TypedDict. Keeping state explicit makes reasoning, testing, and checkpointing straightforward.

```python
# agents/state.py
from typing import TypedDict, Optional, List, Annotated
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # ── Workflow control ──────────────────────────────────────────────
    phase: str                          # supervisor routes on this value
    session_id: str                     # MySQL session UUID
    feedback_iteration: int             # feedback loop counter

    # ── Incident data ─────────────────────────────────────────────────
    incident_id: Optional[str]
    severity: Optional[str]
    anomaly_nodes: List[dict]           # set by Incident Detector
    root_causes: List[dict]             # set by Root Cause Finder
    mitigation_workflows: List[dict]    # set by Root Cause Finder (pre-fetched)
    mitigation_confidence: float        # drives feedback loop
    communications_sent: List[dict]     # reserved for future use
    incident_summary: Optional[str]     # set by Incident Summarizer

    # ── LangGraph message history ─────────────────────────────────────
    messages: Annotated[list, add_messages]   # accumulates via reducer

    # ── Feedback channel ──────────────────────────────────────────────
    feedback_request: Optional[str]     # mitigator → root_cause_finder

    # ── Communication routing ─────────────────────────────────────────
    communication_event: Optional[str]  # "incident_detected" | "root_cause_found" | "mitigation_complete"
    next_phase: Optional[str]           # phase the communicator advances to after printing
```

Key conventions:

- `phase` is the **single source of truth for routing**. The Supervisor never calls an LLM — it just reads `phase`.
- Lists are returned as full replacements by each agent; `messages` uses `add_messages` to accumulate.
- `feedback_iteration` lets the mitigator cap cycles before escalating.
- `communication_event` + `next_phase` are set by every agent before routing to `phase="communicate"`. The Communicator reads both, prints the appropriate banner, then returns `phase=next_phase`.

---

## 3. Supervisor / Orchestrator

The Supervisor is a **deterministic routing function** — no LLM, no tools. It reads `phase` and returns the name of the next node:

```python
# agents/supervisor.py
from langgraph.graph import END
from .state import AgentState

_ROUTING = {
    "detect":     "incident_detector",
    "root_cause": "root_cause_finder",
    "mitigate":   "incident_mitigator",
    "feedback":   "root_cause_finder",    # feedback loop re-runs RCA
    "communicate":"incident_communicator",
    "summarize":  "incident_summarizer",
}

def supervisor(state: AgentState) -> str:
    return _ROUTING.get(state.get("phase", "done"), END)
```

Because it doesn't call an LLM, routing adds no latency and is trivially testable (12 unit tests cover all phase values).

---

## 4. Agent Pipeline Overview

### Happy-path flow

```
trigger.py polls graphserv for active anomalies
    │
    ▼
incident_detector
    │  LLM classifies anomalies → declares incident, creates IncidentTicket in Neo4j
    │  sets: phase="communicate", communication_event="incident_detected", next_phase="root_cause"
    ▼
incident_communicator  ──  🚨 INCIDENT DECLARED banner + partial Qdrant upsert
    │  sets: phase="root_cause"
    ▼
root_cause_finder
    │  Graph traversal + Qdrant RAG (5 collections) + LLM synthesis → hypotheses + workflows
    │  sets: phase="communicate", communication_event="root_cause_found", next_phase="mitigate"
    ▼
incident_communicator  ──  🔍 ROOT CAUSE IDENTIFIED banner + partial Qdrant upsert
    │  sets: phase="mitigate"
    ▼
incident_mitigator
    │  Prints root causes + workflow steps, simulates execution
    │  ┌─ if confidence < threshold AND iter < max ─────────────────────────────┐
    │  │  saves {outcome="low_confidence"} → Qdrant feedback_history             │
    │  │  sets: phase="feedback" → back to root_cause_finder                    │
    │  └─────────────────────────────────────────────────────────────────────────┘
    │  [when confident or max iterations reached]
    │  if resolved after feedback: saves {outcome="resolved"} → Qdrant feedback_history
    │  sets: phase="communicate", communication_event="mitigation_complete", next_phase="summarize"
    ▼
incident_communicator  ──  🔧 MITIGATION EXECUTED banner + partial Qdrant upsert
    │  sets: phase="summarize"
    ▼
incident_summarizer
    │  Composes full summary from state, prints 📋 INCIDENT SUMMARY
    │  Upserts final summary → Qdrant incident_summaries (same point ID, overwrites partials)
    │  sets: phase="done", incident_summary=<text>
    ▼
  END
```

### Feedback loop

```
incident_mitigator  [confidence=0.42, iter=0]
    │  → saves low_confidence episode to Qdrant feedback_history
    │  → phase="feedback"
    ▼
root_cause_finder  (second pass)
    │  reads feedback_request from state
    │  reads feedback_history from Qdrant → avoids dead-end hypotheses
    │  produces revised hypotheses + workflows
    │  → phase="communicate", next_phase="mitigate"
    ▼
incident_communicator → phase="mitigate"
    ▼
incident_mitigator  [confidence=0.83, iter=1]
    │  → saves resolved episode to Qdrant feedback_history
    │  → phase="communicate", next_phase="summarize"
    ▼
  (continues to summarizer)
```

---

## 5. Incident Detector

**File:** `agents/incident_detector.py`
**Model:** Qwen 2.5 7B (`qwen2.5:7b`, `format="json"`)
**MCP tools:** `list_anomalies`, `create_incident_ticket`

**Logic:**

1. Calls `list_anomalies(status="active", limit=50)` via Graph DB MCP.
2. LLM classifies: `{"is_incident": bool, "severity": "SEV1-4", "reasoning": "...", "affected_node_ids": [...]}`
3. If `is_incident=true`: generates `INC-<uuid>`, calls `create_incident_ticket`.
4. Returns routing fields for the Communicator.

```python
async def incident_detector(state: AgentState) -> dict:
    raw = await tool_map["list_anomalies"].ainvoke({"status": "active", "limit": 50})
    anomalies = json.loads(raw)

    response = await llm.ainvoke([SystemMessage(SYSTEM_PROMPT), HumanMessage(json.dumps(anomalies))])
    decision = json.loads(response.content)

    if not decision.get("is_incident"):
        return {"phase": "done", "anomaly_nodes": anomalies}

    incident_id = f"INC-{uuid.uuid4().hex[:8].upper()}"
    await tool_map["create_incident_ticket"].ainvoke({
        "id": incident_id, "severity": decision["severity"],
        "status": "open", "startTime": now_utc(),
    })

    return {
        "phase": "communicate",
        "communication_event": "incident_detected",
        "next_phase": "root_cause",
        "incident_id": incident_id,
        "severity": decision["severity"],
        "anomaly_nodes": anomalies,
        "messages": [AIMessage(...)],
    }
```

**Graceful fallbacks:**
- `list_anomalies` fails → returns `phase="done"` with error log (no crash)
- LLM returns malformed JSON → treated as no-incident, logs error

---

## 6. Incident Communicator

**File:** `agents/incident_communicator.py`
**Model:** none (deterministic — no LLM call)
**Qdrant writes:** `incident_summaries` (partial upsert, deterministic point ID)

Called after each of three events: `incident_detected`, `root_cause_found`, `mitigation_complete`.

**Logic:**

1. Reads `communication_event` and `next_phase` from state.
2. Prints a formatted banner to stdout:

| `communication_event` | Banner |
|---|---|
| `incident_detected` | 🚨 INCIDENT DECLARED — ID, severity, anomaly list, reason, timestamp |
| `root_cause_found` | 🔍 ROOT CAUSE IDENTIFIED — top hypothesis, confidence, workflow count |
| `mitigation_complete` | 🔧 MITIGATION EXECUTED — workflow applied, steps, confidence score |

3. Upserts a rolling partial summary to Qdrant `incident_summaries` (best-effort, never blocks).
4. Returns `phase=next_phase`, clears `communication_event` and `next_phase`.

**Point ID strategy:** `uuid5(NAMESPACE_DNS, "incident-summary:<incident_id>")` — stable across all three calls and the final summarizer upsert, so each call updates the same Qdrant point rather than creating duplicates.

```python
def make_incident_communicator(qdrant_client=None, embeddings=None):
    async def incident_communicator(state: AgentState) -> dict:
        event      = state.get("communication_event", "unknown")
        next_phase = state.get("next_phase", "done")

        # print banner based on event type ...

        if qdrant_client and embeddings:
            await _upsert_summary(qdrant_client, embeddings, state, event)

        return {
            "phase": next_phase,
            "communication_event": None,
            "next_phase": None,
            "messages": [AIMessage(f"[{event}] communicated. Advancing to {next_phase}.")],
        }
    return incident_communicator
```

---

## 7. Root Cause Finder

**File:** `agents/root_cause_finder.py`
**Model:** Llama 3.1 8B (`llama3.1:8b`, `format="json"`)
**MCP tools:** `get_relationships`, `root_cause_analysis`, `blast_radius`, `get_change_tickets`, `get_rca_tickets`

### Graph traversal

For each anomaly node:
1. `get_relationships(fromLabel="Anomaly", fromId=<id>, type="DETECTED_ON")` → finds the topology node the anomaly sits on.
2. `root_cause_analysis(startLabel=<label>, startId=<id>, maxDepth=5)` → downstream traversal, finds related anomalies.
3. `blast_radius(label=<label>, id=<id>)` → count of services depending on the affected node.

### Qdrant semantic search

All five collections are searched using the same natural-language query built from `anomaly_nodes`, `severity`, and any `feedback_request`:

| Collection | Limit | Purpose |
|---|---|---|
| `rca_documents` | 5 | Past RCA write-ups for similar failure patterns |
| `change_context` | 5 | Recent deploys / config changes near incident time |
| `incident_summaries` | 3 | Summaries from previous runs — spot recurring patterns |
| `feedback_history` | 5 | Past low-confidence attempts and resolved outcomes — avoids dead ends |
| `mitigation_workflows` | 3 | Pre-fetched runbooks for the top hypothesis (separate query after LLM) |

### LLM synthesis

All graph evidence + Qdrant hits are concatenated into one prompt. The LLM produces:

```json
{
  "hypotheses": [
    {
      "node_id": "payment-db-001",
      "node_type": "Storage",
      "hypothesis": "Connection pool exhausted after ORM upgrade",
      "evidence": ["ANO-001", "CHG-001"],
      "confidence": 0.9
    }
  ],
  "summary": "Latency spike traced to pgBouncer pool exhaustion..."
}
```

### Feedback pass

When entered via `phase="feedback"`:
- `feedback_request` from state is included in the LLM prompt.
- `feedback_history` from Qdrant tells the LLM which hypotheses previously yielded low confidence.
- The model is explicitly instructed to avoid `outcome="low_confidence"` hypotheses and prefer `outcome="resolved"` patterns.

**Output state:** `phase="communicate"`, `communication_event="root_cause_found"`, `next_phase="mitigate"`, `root_causes`, `mitigation_workflows`, `feedback_request=None`

---

## 8. Incident Mitigator and the Feedback Loop

**File:** `agents/incident_mitigator.py`
**Model:** none (mock — prints to stdout, simulates execution)
**Qdrant writes:** `feedback_history`

### Confidence calculation

- If workflows were found: `confidence = top_workflow["score"]` (cosine similarity from Qdrant search)
- If no workflows but root causes exist: `confidence = max(rc["confidence"] for rc in root_causes)`
- Otherwise: `confidence = 0.0`

### Decision logic

```python
threshold = settings.confidence_threshold    # default 0.75
max_iter  = settings.max_feedback_iterations # default 3

if confidence < threshold and feedback_iteration < max_iter:
    # Save failed attempt so future runs avoid this dead end
    await _save_feedback(qdrant_client, embeddings, state,
                         outcome="low_confidence", confidence=confidence,
                         feedback_msg=feedback_msg)
    return {
        "phase": "feedback",
        "feedback_iteration": feedback_iteration + 1,
        "feedback_request": feedback_msg,
        ...
    }

# Resolved (or first-pass success)
if feedback_iteration > 0:
    # Loop was used — record what ultimately worked
    await _save_feedback(qdrant_client, embeddings, state,
                         outcome="resolved", confidence=confidence)

return {
    "phase": "communicate",
    "communication_event": "mitigation_complete",
    "next_phase": "summarize",
    "mitigation_confidence": confidence,
    ...
}
```

### Qdrant feedback payload

Each feedback episode is written as a **new point** (random UUID) in `feedback_history`, building a log of what failed and what worked:

```json
{
  "outcome": "low_confidence",
  "incident_id": "INC-XXXXXXXX",
  "severity": "SEV2",
  "iteration": 1,
  "confidence": 0.42,
  "hypothesis": "Connection pool exhausted",
  "node_id": "db-001",
  "node_type": "Storage",
  "workflow_title": "Restart DB Connection Pool",
  "feedback_msg": "Confidence 0.42 below threshold. Widen analysis...",
  "recorded_at": "2026-04-06T20:05:00Z"
}
```

### Mock execution output

```
────────────────────────────────────────────────────────────────────────
  INCIDENT MITIGATOR  —  INC-362A3B55  [SEV2]
────────────────────────────────────────────────────────────────────────

[ROOT CAUSES]
  #1  [90% confidence]  payment-db-001 (Storage)
       Hypothesis : Connection pool exhausted after ORM upgrade
       Evidence   : ANO-001, CHG-001

[MITIGATION WORKFLOWS]
  #1  WF-001 — Restart DB Connection Pool  [similarity 0.87]
       Steps:
         1. Confirm active connection count via DB metrics
         2. Restart pgBouncer (pgBouncer reload)
         ...

[CONFIDENCE]  0.87  (threshold: 0.75)

[MOCK EXECUTION — top workflow]
  ✓ step 1: Confirm active connection count  [simulated OK]
  ✓ step 2: Restart pgBouncer               [simulated OK]
```

---

## 9. Incident Summarizer

**File:** `agents/incident_summarizer.py`
**Model:** none (deterministic — built from state, no LLM)
**Qdrant writes:** `incident_summaries` (final authoritative upsert)

**Logic:**

1. Composes a complete plain-text summary from the full `AgentState`:
   - Anomalies detected (count + type, severity, status)
   - Root cause hypotheses ranked by confidence
   - Matched mitigation workflows with steps
   - Confidence score, feedback loop count
2. Prints 📋 INCIDENT SUMMARY to stdout.
3. Upserts to `incident_summaries` using the same deterministic point ID as the Communicator — **overwrites** the three partial updates with the final authoritative record.
4. Sets `phase="done"`, `incident_summary=<text>`.

**Qdrant payload:**

```json
{
  "incident_id": "INC-362A3B55",
  "severity": "SEV2",
  "session_id": "adeecb36-b85a-...",
  "summary": "INCIDENT SUMMARY — INC-362A3B55 [SEV2]\n...",
  "anomaly_count": 1,
  "root_cause_count": 3,
  "workflow_count": 2,
  "mitigation_confidence": 0.87,
  "feedback_iterations": 0,
  "completed_at": "2026-04-06T20:10:00Z"
}
```

**Why no LLM?** The structured data in state already contains everything needed. A deterministic template produces a consistent, auditable record that is fast and never hallucinates.

---

## 10. Qdrant Vector Collections

All five collections use **768-dimensional cosine similarity** vectors via `nomic-embed-text` (Ollama).

| Collection | Written by | Read by | Point ID |
|---|---|---|---|
| `mitigation_workflows` | `seed_data.py` | root_cause_finder | random UUID |
| `rca_documents` | `seed_data.py` | root_cause_finder | random UUID |
| `change_context` | `seed_data.py` | root_cause_finder | random UUID |
| `incident_summaries` | communicator (×3), summarizer (×1) | root_cause_finder | deterministic `uuid5` per incident |
| `feedback_history` | mitigator (low_confidence + resolved) | root_cause_finder | random UUID per episode |

### Cross-run learning

```
Run N completes
    ├── incident_summaries ← communicator partial upserts (event-by-event)
    │                      ← summarizer final upsert (overwrites partials)
    └── feedback_history   ← mitigator low_confidence episodes
                           ← mitigator resolved episodes

Run N+1 — root_cause_finder searches all 5 collections:
    incident_summaries → "last time: connection pool, fixed by restarting pgBouncer"
    feedback_history   → "avoid: CPU hypothesis (low_confidence 0.42)
                          prefer: memory leak hypothesis (resolved 0.83)"
```

### Seeding

Before the first run, populate the static collections:

```bash
cd qdrant_data
python seed_data.py
```

Creates and populates all five collections (including the empty `feedback_history`). Requires Qdrant on `localhost:6333` and Ollama with `nomic-embed-text` on `localhost:11434`.

---

## 11. Assembling the StateGraph

All nodes and edges live in `graph/workflow.py`:

```python
# graph/workflow.py
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from autoincrespagent.agents.supervisor import supervisor
from autoincrespagent.agents.incident_detector import make_incident_detector
from autoincrespagent.agents.incident_communicator import make_incident_communicator
from autoincrespagent.agents.root_cause_finder import make_root_cause_finder
from autoincrespagent.agents.incident_mitigator import make_incident_mitigator
from autoincrespagent.agents.incident_summarizer import make_incident_summarizer
from autoincrespagent.vector.qdrant_search import build_qdrant_client, build_embeddings

def build_graph(all_tools: list, checkpointer=None):
    graph_tools    = [t for t in all_tools if t.name in _GRAPH_DB_TOOL_NAMES]
    qdrant_client  = build_qdrant_client()   # graceful fallback to None if unavailable
    embeddings     = build_embeddings()

    builder = StateGraph(AgentState)

    builder.add_node("incident_detector",
                     make_incident_detector(graph_tools))
    builder.add_node("incident_communicator",
                     make_incident_communicator(qdrant_client=qdrant_client, embeddings=embeddings))
    builder.add_node("root_cause_finder",
                     make_root_cause_finder(graph_tools, qdrant_client=qdrant_client, embeddings=embeddings))
    builder.add_node("incident_mitigator",
                     make_incident_mitigator(qdrant_client=qdrant_client, embeddings=embeddings))
    builder.add_node("incident_summarizer",
                     make_incident_summarizer(qdrant_client=qdrant_client, embeddings=embeddings))

    builder.set_entry_point("incident_detector")

    # All nodes route through the supervisor conditional edge
    for node in ["incident_detector", "root_cause_finder", "incident_mitigator",
                 "incident_communicator", "incident_summarizer"]:
        builder.add_conditional_edges(node, supervisor)

    return builder.compile(checkpointer=checkpointer or MemorySaver())
```

Because the Supervisor is a pure function of `phase`, any node advances the pipeline by setting `phase` on its returned dict. The mitigator's feedback loop is just another phase value — no special graph wiring required.

---

## 12. MySQL Checkpointing

LangGraph state is checkpointed at every node boundary via a custom `MySQLSaver` backed by `aiomysql`. This gives:

- **Resumable workflows** — kill the agent container mid-run and restart from the last checkpoint.
- **Replay** — step through a past incident for debugging.
- **Human-in-the-loop** — pause at any node and require explicit approval before continuing.

```python
# memory/mysql_saver.py
from langgraph.checkpoint.base import BaseCheckpointSaver
import aiomysql, pickle

class MySQLSaver(BaseCheckpointSaver):
    async def aput(self, config, checkpoint, metadata, new_versions):
        async with self.pool.acquire() as conn, conn.cursor() as cur:
            await cur.execute(
                """INSERT INTO checkpoints
                       (thread_id, checkpoint_ns, checkpoint_id, parent_checkpoint_id,
                        type, checkpoint, metadata)
                   VALUES (%s, %s, %s, %s, %s, %s, %s)
                   ON DUPLICATE KEY UPDATE
                       checkpoint = VALUES(checkpoint),
                       metadata   = VALUES(metadata)""",
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

    async def aput_writes(self, config, writes, task_id, task_path: str = ""):
        ...   # stores intermediate writes between checkpoints

    async def aget_tuple(self, config):
        ...   # restores latest checkpoint for a given thread_id
```

**Fallback:** if MySQL is unavailable at startup, `build_graph` falls back to `MemorySaver` (in-process, no persistence). The agent still runs; it just can't resume after a restart.

See the [Database Setup](./02-database-setup.md#3-mysql) document for the `checkpoints` table DDL.

---

## 13. Triggering the Graph

`trigger.py` is the CLI entry point:

```bash
# Run once (single anomaly poll)
python trigger.py

# Poll continuously (every POLL_INTERVAL_SECONDS)
python trigger.py --poll
```

The polling loop queries `list_anomalies` every `POLL_INTERVAL_SECONDS` (default 60). When anomalies are detected it starts a fresh LangGraph thread with a new `session_id` and calls `graph.ainvoke(initial_state, config)`.

### Inject a test anomaly

With no active anomalies, the Detector exits at `phase=done`. Inject one to exercise the full pipeline:

```bash
curl -X POST http://localhost:8080/anomalies \
  -H "Content-Type: application/json" \
  -d '{
    "id": "ANO-TEST-001",
    "type": "latency_spike",
    "severity": "high",
    "status": "active",
    "description": "Test latency spike on payment service"
  }'

python trigger.py
```

### Force a feedback loop (for testing)

```bash
# .env — lower threshold so the first-pass workflow score never clears it
CONFIDENCE_THRESHOLD=0.99

python trigger.py
# mitigator will request 1–3 feedback passes, saving to feedback_history each time
```

### Fallback behaviour

| Service unavailable | Behaviour |
|---|---|
| MySQL | Falls back to MemorySaver — agent runs, no persistence |
| Qdrant | Vector search skipped; agents still produce output from graph evidence |
| graphserv | Detector logs error, returns `phase="done"` cleanly |
| LLM bad JSON | Detector: treated as no-incident. RCA: returns empty hypotheses, advances |

---

## 14. Testing Agents

```bash
cd autoincrespagent
source .venv/bin/activate
pytest tests/unit/ -v
```

### Test coverage

| File | Tests | What's covered |
|---|---|---|
| `test_supervisor.py` | 12 | All phase → node routing; unknown/empty/missing phase → END |
| `test_detector.py` | 18 | No anomalies, no incident, incident declared, MCP failures, LLM errors |
| `test_root_cause_finder.py` | 17 | Factory validation, graph traversal, 5-collection Qdrant search, feedback pass |
| `test_incident_mitigator.py` | 19 | Confidence routing, feedback trigger, max-iterations cap, Qdrant persistence |
| `test_incident_communicator.py` | 11 | All 3 event banners, next_phase routing, Qdrant upsert + failure tolerance |
| `test_incident_summarizer.py` | 12 | Summary content, Qdrant persistence, deterministic point ID matches communicator |
| **Total** | **89** | All offline, < 1 second |

### Unit test approach

- **MCP tools** — mocked with `unittest.mock.AsyncMock`. No running graphserv required.
- **LLM** — injected via `make_<agent>(tools, llm=fake_llm)`. No Ollama required.
- **Qdrant client** — mocked: `client.query_points`, `client.upsert`, `client.get_collections`. No Qdrant required.
- **State** — hand-crafted dicts passed directly to the agent function.

### Example: testing the mitigator feedback save

```python
async def test_saves_low_confidence_to_qdrant():
    mock_client, mock_embeddings = _mock_qdrant()
    agent = make_incident_mitigator(qdrant_client=mock_client, embeddings=mock_embeddings)

    result = await agent(_state(mitigation_workflows=[_wf(score=0.3)], feedback_iteration=0))

    assert result["phase"] == "feedback"
    mock_client.upsert.assert_called_once()
    assert mock_client.upsert.call_args[1]["collection_name"] == "feedback_history"
    assert mock_client.upsert.call_args[1]["points"][0].payload["outcome"] == "low_confidence"
```

### Observability

Every agent emits structured JSON log lines:

```bash
python trigger.py 2>&1 | python -m json.tool
```

Enable LangSmith tracing when you need visual traces of tool calls, LLM prompts, and state transitions:

```bash
LANGCHAIN_TRACING_V2=true LANGCHAIN_API_KEY=<key> python trigger.py
```

---

*See also: [Incident Response Framework](./01-incident-response-framework.md) · [MCP Servers](./04-mcp-servers.md) · [Database Setup](./02-database-setup.md)*
