# Building MCP Servers

Three **Model Context Protocol (MCP)** servers form the Service Layer between the LangGraph agents and the underlying data stores. Each server is a standalone Python process spawned by the agent framework over the stdio transport. MCP gives the agents a **typed tool interface** — every operation (query the graph, search a vector collection, send a Slack message) is declared with a JSON Schema, validated at the boundary, and automatically surfaced to LangChain via `langchain-mcp-adapters`.

This document covers the design and implementation of all three servers, including tool lists, request/response shapes, and how to plug them into the LangGraph agent runtime.

---

## Table of Contents

1. [Why MCP?](#1-why-mcp)
2. [Common Anatomy of an MCP Server](#2-common-anatomy-of-an-mcp-server)
3. [Graph DB MCP Server](#3-graph-db-mcp-server)
4. [Mitigation Services MCP Server](#4-mitigation-services-mcp-server)
5. [Communication MCP Server](#5-communication-mcp-server)
6. [Wiring MCP Servers into LangGraph](#6-wiring-mcp-servers-into-langgraph)
7. [Testing MCP Servers](#7-testing-mcp-servers)

---

## 1. Why MCP?

Agents should never talk to raw databases or HTTP services directly. MCP gives each backend a **single, typed, discoverable** tool surface:

- **Schema enforcement** — every tool has an explicit JSON Schema for its arguments, validated before the call reaches the backend. Malformed LLM outputs are rejected at the MCP boundary, not deep inside the database driver.
- **Discoverability** — agents (and humans) can call `list_tools()` on a running server to see every operation with documentation, parameter types, and enums.
- **Process isolation** — each MCP server is its own Python process. A crash in the mitigation server does not take down the graph server, and you can restart, redeploy, or profile each one independently.
- **Uniform integration** — `langchain-mcp-adapters` converts MCP tool schemas into LangChain-compatible tools, so agents call them with the same tool-calling API they use for any other LangChain tool.
- **Composability** — new backends are added by standing up a new MCP server, not by patching the agent code.

## 2. Common Anatomy of an MCP Server

Every server in this framework follows the same pattern using the Python MCP SDK:

```python
import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("my-mcp-server")

@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="my_tool",
            description="Does a single, well-defined thing.",
            inputSchema={
                "type": "object",
                "properties": {
                    "arg": {"type": "string", "description": "..."},
                },
                "required": ["arg"],
            },
        ),
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "my_tool":
        result = await do_work(arguments["arg"])
        return [types.TextContent(type="text", text=result)]
    raise ValueError(f"unknown tool: {name}")

if __name__ == "__main__":
    asyncio.run(stdio_server(app))
```

Key points:

- `@app.list_tools()` declares the catalogue. Each tool has a name, a human-readable description, and a JSON Schema for its arguments.
- `@app.call_tool()` is the single dispatcher. It receives a validated `arguments` dict and returns a list of `TextContent` (or other content types).
- `stdio_server(app)` starts the server with stdin/stdout framing, which is exactly what LangGraph and other MCP clients expect when they spawn the process.
- Servers are **async-first** — use `httpx.AsyncClient`, `aiomysql`, and the async Qdrant client so multiple tool invocations do not block each other.

## 3. Graph DB MCP Server

**Location:** `mcp_servers/graph_db/server.py`

The Graph DB MCP server wraps the **graphserv** REST API so agents never touch Neo4j directly. It uses `httpx.AsyncClient` for async HTTP calls to graphserv on port 8080, and exposes a tidy catalogue of topology operations.

### 3.1 Tool Catalogue

| Tool | graphserv endpoint | Parameters |
|------|-------------------|------------|
| `list_anomalies` | `GET /api/v1/nodes/Anomaly` | `status`, `limit`, `page` |
| `get_node` | `GET /api/v1/nodes/{label}/{id}` | `label`, `id` |
| `root_cause_analysis` | `POST /api/v1/analysis/root-cause` | `startLabel`, `startId`, `maxDepth`, `anomalyStatus`, `limit` |
| `blast_radius` | `GET /api/v1/analysis/impact` | `label`, `id`, `useTransitive` |
| `get_relationships` | `GET /api/v1/relationships` | `fromLabel`, `fromId`, `type` |
| `create_incident_ticket` | `POST /api/v1/nodes` (label=IncidentTicket) | `id`, `severity`, `status`, `startTime` |
| `link_incident_to_node` | `POST /api/v1/relationships` | `fromLabel`, `fromId`, `toLabel`, `toId`, `type=IMPACTS` |
| `get_rca_tickets` | `GET /api/v1/nodes/RCATicket` | `status`, `limit` |
| `get_change_tickets` | `GET /api/v1/nodes/ChangeTicket` | `status`, `limit` |
| `update_node_status` | `PATCH /api/v1/nodes/{label}/{id}` | `label`, `id`, `status` |

### 3.2 Reference Implementation

```python
# mcp_servers/graph_db/server.py
import asyncio
import httpx
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

GRAPHSERV = "http://graphserv:8080/api/v1"

app = Server("graph-db-mcp")

@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="list_anomalies",
            description="List anomalies from the graph.",
            inputSchema={
                "type": "object",
                "properties": {
                    "status": {"type": "string", "enum": ["open", "resolved", "in_progress"]},
                    "limit":  {"type": "integer", "default": 20},
                },
            },
        ),
        types.Tool(
            name="root_cause_analysis",
            description="Traverse downstream from a node to find active anomalies.",
            inputSchema={
                "type": "object",
                "properties": {
                    "startLabel":    {"type": "string"},
                    "startId":       {"type": "string"},
                    "maxDepth":      {"type": "integer", "default": 5},
                    "anomalyStatus": {"type": "string", "default": "active"},
                    "limit":         {"type": "integer", "default": 50},
                },
                "required": ["startLabel", "startId"],
            },
        ),
        # ...other tools
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    async with httpx.AsyncClient() as client:
        if name == "list_anomalies":
            r = await client.get(f"{GRAPHSERV}/nodes/Anomaly", params=arguments)
            return [types.TextContent(type="text", text=r.text)]
        if name == "root_cause_analysis":
            r = await client.post(f"{GRAPHSERV}/analysis/root-cause", json=arguments)
            return [types.TextContent(type="text", text=r.text)]
        # ...other tool dispatch
    raise ValueError(f"unknown tool: {name}")

if __name__ == "__main__":
    asyncio.run(stdio_server(app))
```

### 3.3 Design Notes

- **Allow-list labels and types** — the underlying graphserv API already validates labels and relationship types, but the MCP tool schemas mirror the allow-list so invalid values are rejected before the HTTP call is even made.
- **Return raw JSON text** — returning `r.text` lets the agent's LLM see a well-formed JSON blob it can parse or quote in prompts. For tools that need structured return types, you can return multiple `TextContent` items or JSON-encoded dicts.
- **Timeouts** — wrap `httpx.AsyncClient(timeout=10.0)` for production to avoid agents hanging on a stuck backend.

## 4. Mitigation Services MCP Server

**Repository:** `mitigationmcpserv/`
**Location:** `mcp_servers/mitigation/server.py`
**Env vars:** `QDRANT_URL`, `OLLAMA_BASE_URL`, `CONFIDENCE_THRESHOLD`

This server provides **semantic search over mitigation workflows** (via Qdrant) and a **stub execution mechanism** for the steps of a chosen workflow. In production you would replace the stubs with real API calls; for the local project, workflow execution logs the action to `logs/mitigation_runs.log` and tracks run state in memory.

### 4.1 Tool Catalogue

| Tool | Description |
|------|-------------|
| `search_mitigation_workflows` | Semantic search in Qdrant — returns top-k workflows with steps and similarity scores. Falls back to 5 built-in mock workflows when Qdrant is empty |
| `execute_mitigation_step` | Stub executor — registers the step in the in-memory run registry and writes a JSON log line to `logs/mitigation_runs.log` |
| `check_mitigation_status` | Returns current execution status of a run: `steps_completed`, `steps_total`, `status` |
| `store_mitigation_feedback` | Embeds and upserts a feedback episode to the Qdrant `feedback_history` collection; returns `{"qdrant_stored": true}` |

### 4.2 Search Flow

1. The Incident Mitigator agent passes the root-cause summary as the query string.
2. `search_mitigation_workflows` calls `OllamaEmbeddings` (`nomic-embed-text`, 768-dim) to embed the query.
3. The server runs `client.search(collection_name="mitigation_workflows", query_vector=…)` with optional payload filters on `trigger_type`, `service_type`, and `severity`.
4. If Qdrant returns results, each hit is returned with its `steps`, workflow name, and cosine similarity score (`source: "qdrant"`).
5. If the collection is empty or Qdrant is unavailable, the server returns 5 built-in mock workflows (`source: "mock"`) so the pipeline keeps running during development.
6. The agent checks the top score against `CONFIDENCE_THRESHOLD` (default `0.75`). Below the threshold, it falls back to the feedback loop described in the [LangGraph Agents](./05-langgraph-agents.md) document.

### 4.3 Stub Execution Model

Each mitigation step is executed through `execute_mitigation_step`, which:

- Registers the step in an **in-memory run registry** (`_RUNS` dict keyed by `run_id`).
- Writes a structured JSON log line to `logs/mitigation_runs.log`.
- Returns `{"status": "executed", "run_id": "...", "step_index": N, "stub": true}`.

`check_mitigation_status` reads the same registry and returns `steps_completed` / `steps_total` so the mitigator can verify the full workflow ran.

`store_mitigation_feedback` embeds the combined query text (hypothesis + node info) using Ollama and upserts a point to the `feedback_history` Qdrant collection with the outcome, confidence, and iteration metadata.

For real deployments, the stub in `execute_mitigation_step` is replaced by a narrow adapter to the corresponding platform (Kubernetes, Nomad, your CI/CD system, etc.).

## 5. Communication MCP Server

**Repository:** `commsmcpserv/`
**Location:** `mcp_servers/comms/server.py`
**Env vars:** `COMMS_LOG_DIR` (default `logs`)

The Communication MCP server provides **unified communication tools** for the Incident Communicator agent. For the local project every channel writes a structured JSON line to a log file and returns a stub delivery ID. The stubs are swapped for real SMTP, Slack webhooks, Teams webhooks, Twilio, or PagerDuty calls when you're ready.

### 5.1 Tool Catalogue

| Tool | Channel | Local Stub | Log file |
|------|---------|------------|----------|
| `send_email` | Email (SMTP) | Appends JSON line, returns `EMAIL-XXXXXXXX` | `logs/email_outbox.log` |
| `send_slack` | Slack webhook | Appends JSON line, returns `SLACK-XXXXXXXX` | `logs/slack_outbox.log` |
| `send_teams` | MS Teams webhook | Appends JSON line, returns `TEAMS-XXXXXXXX` | `logs/teams_outbox.log` |
| `send_sms` | SMS (Twilio) | Appends JSON line, returns `SMS-XXXXXXXX` | `logs/sms_outbox.log` |
| `page_oncall` | PagerDuty | Appends JSON line, returns `PD-XXXXXXXX` | `logs/pagerduty_outbox.log` |
| `update_ticket_comms` | Incident ticket | Appends JSON record of all channels dispatched | `logs/ticket_comms.log` |

Each stub response includes `{"status": "sent", "stub": true, "message_id": "<CHANNEL-XXXXXXXX>", "sent_at": "<ISO timestamp>"}`.

### 5.2 Channel Selection

The Incident Communicator selects channels based on severity:

| Severity | Channels dispatched |
|----------|---------------------|
| SEV1 | `page_oncall` → `send_slack` → `send_email` |
| SEV2 | `send_slack` → `send_teams` → `send_email` |
| SEV3 | `send_email` → `send_teams` |
| SEV4 | `send_email` |

After dispatching all channels, the agent calls `update_ticket_comms` with the list of channels that succeeded. If an individual channel fails, the remaining channels still run — partial dispatch is recorded rather than an all-or-nothing abort.

Each channel tool accepts:

| Parameter | Tools |
|-----------|-------|
| `incident_id`, `severity`, `subject`, `body` | `send_email`, `send_slack`, `send_teams`, `send_sms` |
| `incident_id`, `severity`, `title`, `body` | `page_oncall` |
| `incident_id`, `channels_notified` | `update_ticket_comms` |

The stubs append structured JSON lines to the corresponding log file so that integration tests can assert exactly what was sent and to which channels.

## 6. Wiring MCP Servers into LangGraph

The agent framework spawns every MCP server as a subprocess and wraps its tools using `langchain-mcp-adapters`. As of `langchain-mcp-adapters 0.2.x`, `MultiServerMCPClient` is **not** a context manager — call `get_tools()` directly without `await client.connect()`:

```python
# graph/mcp_client.py
from langchain_mcp_adapters.client import MultiServerMCPClient
from autoincrespagent.config import settings

def build_mcp_client() -> MultiServerMCPClient:
    return MultiServerMCPClient({
        "graph_db": {
            "command": "python",
            "args": ["-m", "mcp_servers.graph_db.server"],
            "transport": "stdio",
            "cwd": settings.graphmcpserv_path,       # ← required: sets sys.path for the subprocess
            "env": {"GRAPHSERV_URL": settings.graphserv_url, "GRAPHSERV_TIMEOUT": "10.0"},
        },
        "mitigation": {
            "command": "python",
            "args": ["-m", "mcp_servers.mitigation.server"],
            "transport": "stdio",
            "cwd": settings.mitigationmcpserv_path,  # ← required
            "env": {
                "QDRANT_URL": f"http://{settings.qdrant_host}:{settings.qdrant_port}",
                "OLLAMA_BASE_URL": settings.ollama_base_url,
                "CONFIDENCE_THRESHOLD": str(settings.confidence_threshold),
            },
        },
        "comms": {
            "command": "python",
            "args": ["-m", "mcp_servers.comms.server"],
            "transport": "stdio",
            "cwd": settings.commsmcpserv_path,        # ← required
            "env": {"COMMS_LOG_DIR": "logs"},
        },
    })
```

> **Why `cwd` is required:** all three MCP packages share the same `mcp_servers.*` top-level namespace. Without `cwd`, Python binds the namespace to the first package it finds and the other two become invisible. Setting `cwd` to each server's own source directory adds that directory to `sys.path` inside the subprocess, so each server resolves its own `mcp_servers` package independently.

Usage in `graph/workflow.py`:

```python
client    = build_mcp_client()
all_tools = await client.get_tools()   # LangChain-compatible Tool objects

_GRAPH_DB_TOOL_NAMES    = {"list_anomalies", "get_node", "root_cause_analysis",
                            "blast_radius", "get_relationships",
                            "create_incident_ticket", "link_incident_to_node",
                            "get_rca_tickets", "get_change_tickets", "update_node_status"}
_MITIGATION_TOOL_NAMES  = {"search_mitigation_workflows", "execute_mitigation_step",
                            "check_mitigation_status", "store_mitigation_feedback"}
_COMMS_TOOL_NAMES       = {"send_email", "send_slack", "send_teams",
                            "send_sms", "page_oncall", "update_ticket_comms"}

graph_tools      = [t for t in all_tools if t.name in _GRAPH_DB_TOOL_NAMES]
mitigation_tools = [t for t in all_tools if t.name in _MITIGATION_TOOL_NAMES]
comms_tools      = [t for t in all_tools if t.name in _COMMS_TOOL_NAMES]
```

Each agent factory then receives only the tools it needs:

```python
make_incident_mitigator(qdrant_client=qdrant_client, embeddings=embeddings,
                        mitigation_tools=mitigation_tools)
make_incident_communicator(qdrant_client=qdrant_client, embeddings=embeddings,
                           comms_tools=comms_tools)
```

### Tool response unwrapping

`langchain-mcp-adapters` returns `list[TextContent]` from `tool.ainvoke()`, not a plain string. Both `incident_mitigator` and `incident_communicator` unwrap this before JSON-parsing:

```python
async def _call_tool(tools, name, args):
    raw = await tool.ainvoke(args)
    if isinstance(raw, list):
        first = raw[0]
        raw = first.text if hasattr(first, "text") else (
            first.get("text", str(first)) if isinstance(first, dict) else str(first)
        )
    return json.loads(raw) if isinstance(raw, str) else raw
```

Because MCP tools look identical to any other LangChain tool, the LLM's tool-calling machinery works without changes.

## 7. Testing MCP Servers

### 7.1 Unit Tests for Agent MCP Integration

Agent-level unit tests mock MCP tools as `AsyncMock` objects — no running MCP servers required:

```python
def _mock_mcp_tool(name: str, response: dict):
    tool = MagicMock()
    tool.name = name
    tool.ainvoke = AsyncMock(return_value=json.dumps(response))
    return tool
```

The tool list is then injected into the agent factory:

```python
agent = make_incident_mitigator(mitigation_tools=_mock_mitigation_tools())
agent = make_incident_communicator(comms_tools=_all_comms_tools())
```

Assertions use the dict directly (not `json.loads`) because `ainvoke` is called with a dict, not a JSON string:

```python
args = tool.ainvoke.call_args[0][0]   # already a dict
assert args["incident_id"] == "INC-TEST"
```

| Test File | Tests | What's covered |
|-----------|-------|----------------|
| `tests/unit/test_incident_mitigator.py` | 31 | MCP tool dispatch, step execution, status check, feedback storage via MCP, fallback to direct Qdrant |
| `tests/unit/test_incident_communicator.py` | 34 | Channel selection by severity, tool arguments, `communications_sent` state, graceful degradation, Qdrant upsert |

### 7.2 MCP Server Unit Tests (within each server repo)

- Mock backends with `httpx` request mocking (`respx`) or pytest `monkeypatch`.
- Assert that each tool translates arguments into the correct HTTP call / Qdrant query / log line.
- Validate input schemas with the Python MCP SDK schema validator.

| Test Target | File | Focus |
|-------------|------|-------|
| Graph DB MCP | `graphmcpserv/tests/test_mcp_graph.py` | Every tool calls the correct graphserv endpoint with correct params |
| Mitigation MCP | `mitigationmcpserv/tests/test_mcp_mitigation.py` | Qdrant search, mock workflow fallback, stub execution writes logs, `source` field |
| Comms MCP | `commsmcpserv/tests/test_mcp_comms.py` | Each channel stub emits a parseable log line with correct stub ID format |

### 7.3 Integration Tests

Integration tests spawn the MCP servers as real subprocesses and call them from LangChain:

```python
client = build_mcp_client()
tools = await client.get_tools()
result = await [t for t in tools if t.name == "list_anomalies"][0].ainvoke({"status": "open", "limit": 5})
assert "ANOM-" in result
```

Combined with the integration fixtures described in the [Incident Response Framework](./01-incident-response-framework.md#7-testing-strategy) document, this exercises the full data-plane path from agent → MCP → graphserv → Neo4j.

---

*See also: [graphserv REST Service](./03-graphserv-rest-service.md) · [LangGraph Agents](./05-langgraph-agents.md)*
