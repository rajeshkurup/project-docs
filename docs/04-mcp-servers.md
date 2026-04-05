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

**Location:** `mcp_servers/mitigation/server.py`

This server provides **semantic search over mitigation workflows** (via Qdrant) and a **stub execution mechanism** for the steps of a chosen workflow. In production you would replace the stubs with real API calls; for the local project, workflow execution logs the action and updates the incident ticket status through the Graph DB MCP.

### 4.1 Tool Catalogue

| Tool | Description |
|------|-------------|
| `search_mitigation_workflows` | Semantic search in Qdrant — returns top-k workflows with steps and confidence scores |
| `execute_mitigation_step` | Stub executor — logs the action and updates the incident ticket via the Graph DB MCP |
| `check_mitigation_status` | Returns the current execution status of a workflow run |
| `store_mitigation_feedback` | Stores feedback (did it work?) back into the Qdrant payload for learning |

### 4.2 Search Flow

1. The Incident Mitigator agent passes the root-cause summary as the query string.
2. `search_mitigation_workflows` calls `OllamaEmbeddings` (`nomic-embed-text`, 768-dim) to embed the query.
3. The server runs `client.search(collection_name="mitigation_workflows", query_vector=…)` with optional payload filters on `trigger_type`, `service_type`, and `severity`.
4. Each hit returns the stored `steps_json`, a workflow name, and a cosine similarity score.
5. The agent checks the top score against `CONFIDENCE_THRESHOLD` (default `0.75`). Below the threshold, it falls back to the feedback loop described in the [LangGraph Agents](./05-langgraph-agents.md) document.

### 4.3 Stub Execution Model

Each mitigation step is executed through `execute_mitigation_step`, which:

- Reads the step's `kind` (`restart_service`, `scale_out`, `rollback_change`, …) from the workflow JSON.
- Writes a human-readable log line to `logs/mitigation_runs.log`.
- Optionally calls back into the Graph DB MCP to update the incident ticket status via `update_node_status`.

For real deployments, each stub is replaced by a narrow adapter to the corresponding platform (Kubernetes, Nomad, your CI/CD system, etc.).

## 5. Communication MCP Server

**Location:** `mcp_servers/comms/server.py`

The Communication MCP server provides **unified communication tools** for the Incident Communicator agent. For the local project every channel writes to a log file and prints to stdout so you can see exactly what would have been sent. The stubs are swapped for real SMTP, Slack webhooks, Teams webhooks, Twilio, or PagerDuty calls when you're ready.

### 5.1 Tool Catalogue

| Tool | Channel | Local Stub |
|------|---------|------------|
| `send_email` | Email (SMTP) | Writes to `logs/email_outbox.log` |
| `send_slack` | Slack webhook | Writes to `logs/slack_outbox.log` |
| `send_teams` | MS Teams webhook | Writes to `logs/teams_outbox.log` |
| `send_sms` | SMS (Twilio) | Writes to `logs/sms_outbox.log` |
| `page_oncall` | PagerDuty | Writes to `logs/pagerduty_outbox.log` |
| `update_ticket_comms` | Incident ticket | Calls the Graph DB MCP to PATCH the ticket |

### 5.2 Channel Selection

The Incident Communicator decides which channels to use based on severity, time of day, and escalation rules. Typical defaults:

- **SEV1** — `page_oncall` + `send_slack` + `send_email`
- **SEV2** — `send_slack` + `send_teams` + `send_email`
- **SEV3** — `send_email` + `send_teams`

Each tool accepts a `subject`/`title`, a `body` (pre-rendered by the LLM), and optional metadata such as `recipients` or `severity`. The stubs simply append structured JSON lines to the corresponding log file so that integration tests can assert what was sent.

## 6. Wiring MCP Servers into LangGraph

The agent framework spawns every MCP server as a subprocess and wraps its tools using `langchain-mcp-adapters`:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "graph_db":   {"command": "python", "args": ["-m", "mcp_servers.graph_db.server"],   "transport": "stdio"},
    "mitigation": {"command": "python", "args": ["-m", "mcp_servers.mitigation.server"], "transport": "stdio"},
    "comms":      {"command": "python", "args": ["-m", "mcp_servers.comms.server"],      "transport": "stdio"},
})

await client.connect()
all_tools = await client.get_tools()   # LangChain-compatible Tool objects
```

Each agent then picks the tools it needs:

```python
graph_tools      = [t for t in all_tools if t.name.startswith(("list_", "get_", "root_cause", "blast_"))]
mitigation_tools = [t for t in all_tools if "mitigation" in t.name]
comms_tools      = [t for t in all_tools if t.name.startswith(("send_", "page_"))]

llm = get_llm("root_cause_finder").bind_tools(graph_tools)
```

Because MCP tools look identical to any other LangChain tool, the LLM's tool-calling machinery works without changes.

## 7. Testing MCP Servers

### 7.1 Unit Tests

- Mock backends with `httpx` request mocking (`respx`) or pytest `monkeypatch`.
- Assert that each tool translates arguments into the correct HTTP call / Qdrant query / log line.
- Validate input schemas with the Python MCP SDK schema validator.

| Test Target | File | Focus |
|-------------|------|-------|
| Graph DB MCP | `tests/test_mcp_graph.py` | Every tool calls the correct graphserv endpoint with correct params |
| Mitigation MCP | `tests/test_mcp_mitigation.py` | Qdrant search returns top-k with expected payloads; stub execution writes logs |
| Comms MCP | `tests/test_mcp_comms.py` | Each channel stub emits a parseable log line |

### 7.2 Integration Tests

Integration tests spawn the MCP servers as real subprocesses and call them from LangChain:

```python
client = MultiServerMCPClient({"graph_db": {...}})
await client.connect()
tools = await client.get_tools()
anomalies = await [t for t in tools if t.name == "list_anomalies"][0].ainvoke({"status": "open", "limit": 5})
assert "ANOM-" in anomalies
```

Combined with the integration fixtures described in the [Incident Response Framework](./01-incident-response-framework.md#7-testing-strategy) document, this exercises the full data-plane path from agent → MCP → graphserv → Neo4j.

---

*See also: [graphserv REST Service](./03-graphserv-rest-service.md) · [LangGraph Agents](./05-langgraph-agents.md)*
