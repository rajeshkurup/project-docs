# AI Agentic Incident Response Framework — Documentation

Welcome to the documentation for the **AI Agentic Incident Response Framework** — a multi-agent AI system that automates infrastructure incident response end-to-end, built on graphserv, Neo4j, LangGraph, Ollama, Qdrant, and MySQL.

This site is a structured walkthrough of every layer in the stack: the framework itself, the databases it runs on, the RESTful graph service, the MCP servers that expose typed tools to agents, and the LangGraph agents that do the work.

---

## Documentation Map

1. **[Incident Response Framework & Plan](./01-incident-response-framework.md)** — architecture, goals, technology stack, component design, phased development plan, testing strategy, and Docker deployment.
2. **[Database Setup, Schemas & Topology](./02-database-setup.md)** — installation, verification, schema creation, and verification for Neo4j, MySQL, and Qdrant, plus the full Neo4j topology schema.
3. **[graphserv RESTful Service](./03-graphserv-rest-service.md)** — Go-based REST API wrapping Neo4j: architecture, graph schema, configuration, full API reference, Docker, testing, and project layout.
4. **[Building MCP Servers](./04-mcp-servers.md)** — designing, implementing, and wiring the Graph DB, Mitigation, and Comms MCP servers so agents can call typed tools.
5. **[Building Agents with LangGraph](./05-langgraph-agents.md)** — shared state, Supervisor routing, the six specialist agents, the mitigation feedback loop, MySQL checkpointing, and testing.

For a full history of changes across all repositories in this project, see the **[Changelog](../../CHANGELOG.md)**.

---

## Quick Start

1. Follow the **[Database Setup](./02-database-setup.md)** guide to provision Neo4j, MySQL, and Qdrant.
2. Start **[graphserv](./03-graphserv-rest-service.md)** and verify `GET /health` responds.
3. Stand up the three **[MCP servers](./04-mcp-servers.md)** (graph_db, mitigation, comms).
4. Run the **[LangGraph agent pipeline](./05-langgraph-agents.md)** using `python trigger.py --poll`.
5. Watch logs or enable LangSmith tracing to observe the full incident lifecycle.

For a single-file narrative covering the architecture, phased plan, and deployment model, start with the **[Incident Response Framework](./01-incident-response-framework.md)** document.

---

## Publishing as GitHub Pages

These documents are written as plain Markdown and can be published as GitHub Pages in a few ways:

- **Jekyll (default)** — GitHub Pages will render this folder automatically. Commit the `docs/` folder, then in the repository's *Settings → Pages*, set the source to "Deploy from a branch" and the folder to `/docs`.
- **Just the Docs / MkDocs Material** — drop in a `_config.yml` (Jekyll) or `mkdocs.yml` (MkDocs) to enable theming, search, and a nav bar. All cross-links in these documents are relative and will work with either setup.

All internal links use relative paths, so moving the folder to `docs/`, `site/`, or any subdirectory at the root of the repository will work without edits.
