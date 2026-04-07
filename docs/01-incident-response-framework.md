# AI Agentic Incident Response Framework

> A detailed design, development, and deployment blueprint for a multi-agent AI system that automates infrastructure incident response end-to-end — built on **graphserv · Neo4j · LangGraph · Ollama · Qdrant · MySQL**.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Goals](#2-system-goals)
3. [Architecture Overview](#3-architecture-overview)
4. [Technology Stack](#4-technology-stack)
5. [Component Design](#5-component-design)
6. [Phase-by-Phase Development Plan](#6-phase-by-phase-development-plan)
7. [Testing Strategy](#7-testing-strategy)
8. [Docker Deployment](#8-docker-deployment)
9. [Project Directory Structure](#9-project-directory-structure)
10. [Learning Roadmap](#10-learning-roadmap)

---

## 1. Executive Summary

This framework is a comprehensive blueprint for building a multi-agent AI system that automates infrastructure incident response end-to-end. It is grounded in **graphserv**, an existing Go-based REST service that manages a Neo4j topology graph, and extends it with a Python-based agentic layer orchestrated by **LangGraph**. Every component runs locally via Docker Compose, making this a fully self-contained hobby and learning project that can scale to production patterns.

The system detects anomalies, correlates them into incidents, performs graph- and RAG-driven root cause analysis, selects and executes mitigation runbooks, communicates out through multiple channels, and finally summarises each incident back into the vector store for future recall.

---

## 2. System Goals

The framework is designed to:

- Automatically detect anomalies from the Neo4j topology graph and determine whether they constitute actionable incidents.
- Perform AI-driven root cause analysis using graph traversal, historical RCAs, and change tickets.
- Identify and execute mitigation workflows retrieved from a semantic vector database.
- Communicate incidents via Email, Slack, Teams, SMS, and PagerDuty (stubbed locally for development).
- Summarise each incident and store learnings back into the vector DB for future recall.
- Implement a feedback loop so the mitigator can request deeper root cause context.
- Persist all agent sessions and memory in MySQL with LangGraph checkpointing for replay and recovery.

---

## 3. Architecture Overview

### 3.1 Layered Architecture

The system is split into three distinct layers:

- **Data Layer** — Neo4j (graph database), MySQL (sessions, checkpoints, memory), and Qdrant (vector database).
- **Service Layer** — graphserv (existing Go REST API) and three MCP servers that expose tool interfaces to the agents.
- **Agentic Layer** — a LangGraph state graph containing six specialised agents orchestrated by a Supervisor node.

```
┌─────────────────────────────────────────────────────────────────┐
│                AGENTIC LAYER (Python / LangGraph)               │
│                                                                 │
│   ┌─────────────┐   routes   ┌─────────────────────────────┐    │
│   │  Supervisor │ ─────────▶ │  Incident Detector          │    │
│   │ Orchestrator│ ◀───────── │  Root Cause Finder          │    │
│   │             │            │  Incident Mitigator  ◀──┐   │    │
│   │             │            │  Incident Communicator  │   │    │
│   │             │            │  Incident Summarizer    │   │    │
│   └─────────────┘            └─────────────┬───────────┘   │    │
│                        feedback loop (low confidence) ─────┘    │
├─────────────────────────────────────────────────────────────────┤
│                   SERVICE LAYER (MCP Servers)                   │
│   ┌───────────────┐  ┌──────────────────┐  ┌────────────────┐   │
│   │ Graph DB MCP  │  │ Mitigation MCP   │  │  Comms MCP     │   │
│   │ (graphserv    │  │ (workflows +     │  │  (Email/Slack/ │   │
│   │  REST client) │  │  Qdrant search)  │  │   PagerDuty)   │   │
│   └──────┬────────┘  └────────┬─────────┘  └───────┬────────┘   │
├──────────┼────────────────────┼────────────────────┼────────────┤
│                         DATA LAYER                              │
│   ┌──────▼────────┐   ┌───────▼──────────┐   ┌─────▼────────┐   │
│   │ graphserv(Go) │   │ Qdrant Vector DB │   │ MySQL        │   │
│   │   + Neo4j     │   │  + Ollama embeds │   │ + Memory     │   │
│   └───────────────┘   └──────────────────┘   └──────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Agent Interaction Flow

The Supervisor receives a trigger — a scheduled poll of graphserv — and routes to the **Incident Detector**. When an incident is confirmed, the pipeline routes to the **Incident Communicator** first (to print a notification), then to the **Root Cause Finder**, then back through the Communicator again, and on to the **Incident Mitigator**. If the Mitigator's confidence score falls below a configurable threshold it sends control back to the Root Cause Finder (with a feedback message and Qdrant feedback history) for a deeper analysis pass. Once confidence clears the threshold the pipeline goes through the Communicator a third time and ends at the **Incident Summarizer**, which writes the final incident record to Qdrant.

The Incident Communicator acts as a cross-cutting notification layer: it is invoked by the Supervisor after each of the three main phase transitions (`incident_detected`, `root_cause_found`, `mitigation_complete`) and also upserts a rolling partial summary to Qdrant `incident_summaries` after each event.

---

## 4. Technology Stack

| Component | Technology | Version / Notes |
|-----------|------------|-----------------|
| Orchestration framework | LangGraph (Python) | ≥ 1.0 — stateful multi-agent graph |
| LLM integration | LangChain (Python) | ≥ 1.0 — tools, prompts, chains |
| Local LLM runtime | Ollama | Llama 3.1 8B / Qwen 2.5 7B (tool-calling) |
| Embedding model | Ollama `nomic-embed-text` | 768-dim, runs locally |
| Graph database | Neo4j Community 5.x | Accessed via graphserv Go REST API |
| Vector database | Qdrant | Docker image — 5 collections |
| Relational database | MySQL 8 | Sessions, checkpoints, memory bank |
| MCP framework | Python MCP SDK (`mcp`) | ≥ 1.0, stdio transport |
| MCP client | `langchain-mcp-adapters` | ≥ 0.2, converts MCP tools → LangChain tools |
| Graph service | graphserv (existing Go API) | Wraps Neo4j, runs on `:8080` |
| Containerisation | Docker + Docker Compose | All services in one compose file |
| Testing | pytest + pytest-asyncio | 89 unit tests, all offline |
| Async HTTP client | `httpx` | Used by Graph MCP server |
| MySQL async client | `aiomysql` | LangGraph checkpoint saver |
| Qdrant client | `qdrant-client` | ≥ 1.7, `AsyncQdrantClient` |

---

## 5. Component Design

### 5.1 LLM Handling Layer

All LLM inference runs through **Ollama**, which serves models via a local HTTP API on port `11434`. LangChain's `ChatOllama` class is used as the LLM backend for all agents, and a shared factory module centralises model selection, temperature settings, and structured-output configuration.

**Model selection strategy**

| Agent | Recommended Model | Reason |
|-------|-------------------|--------|
| Incident Detector | Qwen 2.5 7B | Strong structured JSON / tool-calling output |
| Root Cause Finder | Llama 3.1 8B | Multi-hop reasoning and summarisation |
| Incident Mitigator | Qwen 2.5 7B | Reliable tool calling for workflow selection |
| Incident Communicator | Mistral 7B | Fluent natural language for messages |
| Incident Summarizer | Llama 3.1 8B | Coherent long-form summaries |
| Supervisor | Llama 3.1 8B | Fast, deterministic routing |

**LLM factory** (`llm/factory.py`):

```python
from langchain_ollama import ChatOllama

def get_llm(agent_name: str, temperature: float = 0.1):
    MODEL_MAP = {
        "incident_detector":  "qwen2.5:7b",
        "root_cause_finder":  "llama3.1:8b",
        "incident_mitigator": "qwen2.5:7b",
        "communicator":       "mistral:7b",
        "summarizer":         "llama3.1:8b",
        "supervisor":         "llama3.1:8b",
    }
    return ChatOllama(
        model=MODEL_MAP.get(agent_name, "llama3.1:8b"),
        temperature=temperature,
        base_url="http://ollama:11434",
        format="json" if agent_name in ["incident_detector", "incident_mitigator"] else None,
    )
```

**Embedding model** — `nomic-embed-text` runs via Ollama and produces 768-dimensional vectors. The embedding function is shared across all components that write to or read from Qdrant:

```python
from langchain_ollama import OllamaEmbeddings
embeddings = OllamaEmbeddings(model="nomic-embed-text", base_url="http://ollama:11434")
```

### 5.2 MySQL — Sessions & Memory Bank

MySQL stores structured session data and agent memory. LangGraph's built-in checkpointing is configured to use a custom MySQL checkpoint saver, giving session replay and the ability to resume incomplete incident workflows. A separate `agent_memory` table holds long-term facts agents accumulate across sessions, and a `feedback_log` captures each feedback-loop cycle between the Mitigator and the Root Cause Finder.

Full schema details are in the [Database Setup](./02-database-setup.md) document.

### 5.3 Qdrant — Vector Database

Qdrant runs as a Docker container on port `6333` with persistent storage mounted to a local volume. Five collections are used, all with 768-dimension cosine similarity vectors via `nomic-embed-text`:

| Collection | Content | Written by | Read by |
|---|---|---|---|
| `mitigation_workflows` | Step-by-step runbooks for fixing incidents | `seed_data.py` | Root Cause Finder |
| `rca_documents` | Historical RCA reports and post-mortems | `seed_data.py` | Root Cause Finder |
| `change_context` | Recent change tickets affecting infrastructure nodes | `seed_data.py` | Root Cause Finder |
| `incident_summaries` | Rolling + final summaries of each incident run | Communicator (×3), Summarizer (×1) | Root Cause Finder |
| `feedback_history` | Low-confidence and resolved feedback episodes | Incident Mitigator | Root Cause Finder |

The `incident_summaries` and `feedback_history` collections grow at runtime — every completed incident run writes to them, making the system progressively smarter. All other collections are seeded once via `qdrant_data/seed_data.py`.

Summary upserts use a deterministic `uuid5` point ID so the Communicator's three partial updates are overwritten by the Summarizer's final record — one point per incident. Feedback history uses random UUIDs so every episode is preserved as a unique point.

### 5.4 MCP Servers

Three MCP servers are implemented using the Python MCP SDK. Each is a standalone Python process spawned by the LangGraph agent layer via stdio transport. LangChain's `langchain-mcp-adapters` library converts MCP tool schemas into LangChain-compatible tools that agents can call like any other. See the [MCP Servers](./04-mcp-servers.md) document for full details.

### 5.5 LangGraph Agent Design

The entire agentic system is a LangGraph `StateGraph`. A shared `AgentState` TypedDict is passed through every node. Each agent reads from state, performs its work (calling tools, LLM inference, or DB writes), and returns a partial state update. The Supervisor uses a conditional edge to route based on the current `phase`. See the [LangGraph Agents](./05-langgraph-agents.md) document for full details on state, nodes, and the feedback loop.

---

## 6. Phase-by-Phase Development Plan

Each phase is independently testable before the next begins. Development proceeds bottom-up: infrastructure first, then MCP servers, then individual agents, then the full orchestrated graph.

### Phase 1 — Infrastructure Setup (~2 days)

- Write `docker-compose.yml` with all seven services (neo4j, graphserv, mysql, qdrant, ollama, mcp_graph, agents).
- Apply the Neo4j schema (see [Database Setup](./02-database-setup.md)).
- Create the MySQL schema from the DDL in Section 5.2.
- Initialise Qdrant collections using `vector/setup.py`.
- Pull Ollama models: `llama3.1:8b`, `qwen2.5:7b`, `mistral:7b`, `nomic-embed-text`.
- Seed `mitigation_workflows` with 5–10 sample runbook YAML files.
- Verify graphserv health endpoint responds at `http://localhost:8080/health`.

### Phase 2 — Graph DB MCP Server (~2 days)

- Scaffold `mcp_servers/graph_db/server.py` with all ten tools.
- Write unit tests for each tool using httpx mocking.
- Integration test: spawn the server as a subprocess and call `list_anomalies` against a running graphserv.
- Validate tool schemas using the Python MCP SDK validation utilities.

### Phase 3 — Mitigation & Comms MCP Servers (~2 days)

- Build `mcp_servers/mitigation/server.py` with Qdrant search tools.
- Build `mcp_servers/comms/server.py` with stub channel senders (log-to-file).
- Test Qdrant semantic search with sample mitigation queries.
- Verify MCP tools are callable via `langchain-mcp-adapters`.

### Phase 4 — Core Agents: Detector & Root Cause Finder (~3 days)

- Implement `agents/state.py` with the full `AgentState` TypedDict.
- Implement `agents/incident_detector.py` — connect Graph DB MCP, write LLM prompt, parse structured output.
- Implement `agents/root_cause_finder.py` — graph traversal plus Qdrant RAG plus LLM synthesis.
- Write the MySQLSaver checkpoint adapter (`memory/mysql_saver.py`).
- Unit test each agent node with mocked MCP tools and a mocked LLM.
- Integration test: inject a test anomaly into Neo4j and run detector → root cause finder.

### Phase 5 — Mitigator with Feedback Loop (~3 days)

- Implement `agents/incident_mitigator.py` with confidence threshold and feedback routing.
- Implement the feedback loop as a conditional edge routing mitigator → root_cause_finder.
- Write `feedback_log` inserts in MySQL when the loop triggers.
- Force low confidence and verify the loop routes back and then resolves on a second attempt.
- Tune `MAX_ITERATIONS` and `CONFIDENCE_THRESHOLD` constants.

### Phase 6 — Communicator & Summarizer (~2 days)

- Implement `agents/incident_communicator.py` — message drafting and multi-channel dispatch.
- Implement `agents/incident_summarizer.py` — story synthesis, Qdrant write, MySQL update.
- Test communicator output files in `logs/`.
- Test summarizer produces well-formed Qdrant payloads.

### Phase 7 — Full Graph Assembly & Orchestration (~2 days)

- Assemble the full `StateGraph` in `graph/workflow.py`: add all nodes and edges.
- Add the Supervisor conditional edge for routing.
- Wire MySQL checkpointing to the graph.
- Run an end-to-end test with a real Neo4j anomaly.
- Add a simple trigger CLI (`python trigger.py --poll`) that polls graphserv every 60 seconds.

### Phase 8 — Testing & Refinement (~2 days)

- Write a pytest integration test suite covering the full pipeline with fixtures.
- Add structured logging and observability to every agent.
- Tune LLM prompts based on observed outputs.
- Validate Docker Compose resource limits for a laptop.
- Document the runbook for adding new mitigation workflows.

---

## 7. Testing Strategy

### 7.1 Unit Testing

Each agent module and MCP server is unit-tested in isolation. LLM calls are mocked using LangChain's `FakeListChatModel`. MCP tool calls are mocked using pytest `monkeypatch`. MySQL and Qdrant clients are replaced with in-memory fakes.

| Test Target | Test File | Key Assertions |
|-------------|-----------|----------------|
| Incident Detector | `tests/test_detector.py` | Returns `incident=False` for no anomalies; `True` with severity for anomalies |
| Root Cause Finder | `tests/test_rcf.py` | Combines graph result + Qdrant hits into coherent hypothesis |
| Incident Mitigator | `tests/test_mitigator.py` | Low confidence triggers feedback; high confidence returns steps |
| Feedback Loop | `tests/test_feedback.py` | Loop terminates at `MAX_ITERATIONS`; `feedback_log` row written each cycle |
| Graph DB MCP | `tests/test_mcp_graph.py` | Each tool calls the correct graphserv endpoint with correct params |
| MySQL Saver | `tests/test_mysql_saver.py` | `aput` stores checkpoint; `aget_tuple` restores identical state |
| Qdrant Search | `tests/test_qdrant.py` | Returns top-k results with expected payload fields |

### 7.2 Integration Testing

Integration tests run against the full Docker Compose stack (`docker compose --profile test up`). A fixtures module seeds Neo4j with a known topology, inserts test anomalies, and verifies that the pipeline produces the expected MySQL session record and Qdrant summary entry.

- `tests/integration/test_full_pipeline.py` — end-to-end happy path
- `tests/integration/test_feedback_loop.py` — forced low-confidence scenario
- `tests/integration/test_resume.py` — kill agent mid-run, restart, verify checkpoint recovery

### 7.3 Observability

Every agent emits structured JSON log lines. During development a simple log tail is enough:

```bash
docker compose logs -f agents | python -m json.tool
```

LangSmith tracing can be enabled when you want visual traces of agent steps:

```bash
LANGCHAIN_TRACING_V2=true LANGCHAIN_API_KEY=<key> python trigger.py
```

---

## 8. Docker Deployment

### 8.1 `docker-compose.yml` Structure

All seven services are defined in a single `docker-compose.yml` in the project root. An `incident-net` bridge network connects them, and persistent volumes store Neo4j data, MySQL data, Qdrant storage, and Ollama models across restarts.

```yaml
version: '3.9'
networks:
  incident-net:
    driver: bridge
volumes:
  neo4j_data:
  mysql_data:
  qdrant_data:
  ollama_models:
services:
  neo4j:
    image: neo4j:5-community
    networks: [incident-net]
    ports: ['7474:7474', '7687:7687']
    environment:
      NEO4J_AUTH: neo4j/password123
      NEO4J_PLUGINS: '["apoc"]'
    volumes: ['neo4j_data:/data']
    deploy: { resources: { limits: { memory: 1G } } }

  graphserv:
    image: rajeshkurup77/graphserv:latest
    networks: [incident-net]
    ports: ['8080:8080']
    environment:
      NEO4J_URI: bolt://neo4j:7687
      NEO4J_PASSWORD: password123
    depends_on: [neo4j]
    deploy: { resources: { limits: { memory: 256M } } }

  mysql:
    image: mysql:8
    networks: [incident-net]
    ports: ['3306:3306']
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: incident_response
    volumes:
      - mysql_data:/var/lib/mysql
      - ./sql/schema.sql:/docker-entrypoint-initdb.d/schema.sql
    deploy: { resources: { limits: { memory: 512M } } }

  qdrant:
    image: qdrant/qdrant:latest
    networks: [incident-net]
    ports: ['6333:6333']
    volumes: ['qdrant_data:/qdrant/storage']
    deploy: { resources: { limits: { memory: 512M } } }

  ollama:
    image: ollama/ollama:latest
    networks: [incident-net]
    ports: ['11434:11434']
    volumes: ['ollama_models:/root/.ollama']
    deploy: { resources: { limits: { memory: 8G } } }

  agents:
    build: ./agent_framework
    networks: [incident-net]
    environment:
      MYSQL_URL: mysql+aiomysql://root:rootpass@mysql/incident_response
      QDRANT_HOST: qdrant
      GRAPHSERV_URL: http://graphserv:8080
      OLLAMA_URL: http://ollama:11434
      CONFIDENCE_THRESHOLD: '0.75'
      MAX_FEEDBACK_ITERATIONS: '3'
      POLL_INTERVAL_SECONDS: '60'
    depends_on: [mysql, qdrant, ollama, graphserv]
    volumes: ['./logs:/app/logs']
    deploy: { resources: { limits: { memory: 2G } } }
```

### 8.2 Memory Budget for a 16 GB Laptop

| Service | Memory Limit | Notes |
|---------|--------------|-------|
| Ollama + one 7B model | 8 GB | Biggest consumer; only one model loaded at a time |
| Neo4j | 1 GB | Community edition, typical dev usage well under 1 GB |
| MySQL | 512 MB | Session data is lightweight |
| Qdrant | 512 MB | In-memory index, four small collections |
| agents (Python) | 2 GB | LangGraph state + httpx + qdrant-client + SQLAlchemy |
| graphserv (Go) | 256 MB | Static binary, typically uses well under 128 MB |
| **Total** | **~12 GB** | Leaves ~4 GB for OS on a 16 GB laptop |

> **Tip** — On Docker Desktop for Mac, set *Settings → Resources → Memory* to 13 GB and allocate all available cores. On Linux, ensure swap is enabled (`sudo swapon --show`) as a safety net.
>
> **Ollama tip** — Models load lazily. Pull all of them before running:
>
> ```bash
> ollama pull llama3.1:8b && ollama pull qwen2.5:7b && \
> ollama pull mistral:7b && ollama pull nomic-embed-text
> ```
>
> Total disk usage is approximately 20 GB.

---

## 9. Project Directory Structure

```
incident-response/
├── docker-compose.yml                       # All 6 services
├── graphserv/                               # Go REST API (Neo4j wrapper)
├── graphmcpserv/                            # Graph DB MCP server (Python)
│   └── mcp_servers/graph_db/server.py
├── qdrant_data/
│   └── seed_data.py                         # Seeds all 5 Qdrant collections
└── autoincrespagent/                        # LangGraph agent package
    ├── Dockerfile
    ├── pyproject.toml
    ├── .env.example
    ├── trigger.py                           # CLI entry point (poll loop)
    ├── autoincrespagent/
    │   ├── config.py                        # pydantic-settings — reads .env
    │   ├── llm/
    │   │   └── factory.py                   # get_llm(agent_name) → ChatOllama
    │   ├── agents/
    │   │   ├── state.py                     # AgentState TypedDict
    │   │   ├── supervisor.py                # Deterministic phase router
    │   │   ├── incident_detector.py         # Phase 1 — anomaly classification
    │   │   ├── incident_communicator.py     # Cross-phase — print + Qdrant upsert
    │   │   ├── root_cause_finder.py         # Phase 2 — graph + RAG + LLM
    │   │   ├── incident_mitigator.py        # Phase 3 — mock execution + feedback save
    │   │   └── incident_summarizer.py       # Phase 5 — summary + Qdrant upsert
    │   ├── graph/
    │   │   ├── mcp_client.py                # MultiServerMCPClient factory
    │   │   └── workflow.py                  # StateGraph assembly
    │   ├── memory/
    │   │   └── mysql_saver.py               # LangGraph MySQL checkpoint saver
    │   └── vector/
    │       └── qdrant_search.py             # Shared async Qdrant search helper
    ├── sql/
    │   └── schema.sql                       # MySQL DDL
    └── tests/
        └── unit/
            ├── test_supervisor.py
            ├── test_detector.py
            ├── test_root_cause_finder.py
            ├── test_incident_mitigator.py
            ├── test_incident_communicator.py
            └── test_incident_summarizer.py
```

---

## 10. Learning Roadmap

Since this is also a learning project, here is the sequence of core concepts you will encounter and where to read more.

| Phase | Concept | Resource |
|-------|---------|----------|
| 1 — Infrastructure | Docker Compose networking, volumes, health checks | docs.docker.com/compose |
| 2 — Graph MCP | MCP protocol: tools, schemas, stdio transport | modelcontextprotocol.io/docs |
| 2 — Graph MCP | `httpx` async HTTP client in Python | www.python-httpx.org |
| 3 — Mitigation MCP | Qdrant vector search, payload filtering, scored results | qdrant.tech/documentation |
| 3 — Mitigation MCP | Text embedding, cosine similarity, RAG fundamentals | langchain.com/docs/concepts |
| 4 — Core Agents | LangGraph `StateGraph`, nodes, edges, state reducers | langchain-ai.github.io/langgraph |
| 4 — Core Agents | LangChain tool-calling, structured output, `ChatOllama` | python.langchain.com/docs |
| 4 — Core Agents | MySQL async with aiomysql and SQLAlchemy | docs.sqlalchemy.org |
| 5 — Feedback Loop | Conditional edges, `interrupt_after`, breakpoints | langchain-ai.github.io/langgraph |
| 7 — Full Graph | LangGraph checkpointing: replay, fault tolerance, HITL | langchain-ai.github.io/langgraph |
| 8 — Testing | pytest-asyncio for async agent tests | docs.pytest.org |

> **Suggested build order for maximum learning:** (1) get graphserv talking to Neo4j in Docker, (2) build the Graph DB MCP server and call it from a Python script, (3) build one simple LangGraph single-agent that calls MCP tools, (4) add the MySQL checkpoint saver and verify session persistence, (5) expand to the full six-agent graph one agent at a time. Each step should leave you with a complete, working system.

---

*See also: [Database Setup](./02-database-setup.md) · [graphserv REST Service](./03-graphserv-rest-service.md) · [MCP Servers](./04-mcp-servers.md) · [LangGraph Agents](./05-langgraph-agents.md)*
