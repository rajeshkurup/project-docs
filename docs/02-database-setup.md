# Database Setup, Schemas, and Topology

This document is the definitive reference for the three databases powering the AI Agentic Incident Response Framework:

- **Neo4j** — the topology graph (applications, storage, network, tickets, anomalies)
- **MySQL** — sessions, LangGraph checkpoints, agent memory, feedback log
- **Qdrant** — vector database for mitigation workflows, RCA documents, incident summaries, and change context

Every database is covered with installation, verification, schema creation, and schema verification steps so the stack can be reproduced from scratch on any developer laptop.

---

## Table of Contents

1. [Neo4j](#1-neo4j)
2. [Neo4j Topology Schema](#2-neo4j-topology-schema)
3. [MySQL](#3-mysql)
4. [Qdrant](#4-qdrant)
5. [End-to-End Verification Checklist](#5-end-to-end-verification-checklist)

---

## 1. Neo4j

### 1.1 Installation

**macOS (Homebrew):**

```bash
brew install neo4j
brew services start neo4j
```

Or run it in the foreground:

```bash
neo4j console
```

The Neo4j Browser is available at `http://localhost:7474`. Default credentials are `neo4j / neo4j`; you are prompted to change the password on first login.

**Docker (recommended for the compose stack):**

```bash
docker run -d \
  --name neo4j \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/your-password \
  neo4j:5
```

### 1.2 Verify Installation

```bash
brew services list | grep neo4j          # macOS Homebrew
curl http://localhost:7474                # REST responds
```

Inside the browser, run:

```cypher
RETURN 1 AS alive;
```

A single row with `alive = 1` confirms the instance is healthy.

### 1.3 Create Constraints

Uniqueness constraints are enforced on the `id` property of every node label:

```cypher
CREATE CONSTRAINT application_id IF NOT EXISTS FOR (a:Application)    REQUIRE a.id IS UNIQUE;
CREATE CONSTRAINT storage_id     IF NOT EXISTS FOR (s:Storage)        REQUIRE s.id IS UNIQUE;
CREATE CONSTRAINT network_id     IF NOT EXISTS FOR (n:Network)        REQUIRE n.id IS UNIQUE;
CREATE CONSTRAINT incident_id    IF NOT EXISTS FOR (i:IncidentTicket) REQUIRE i.id IS UNIQUE;
CREATE CONSTRAINT change_id      IF NOT EXISTS FOR (c:ChangeTicket)   REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT rca_id         IF NOT EXISTS FOR (r:RCATicket)      REQUIRE r.id IS UNIQUE;
CREATE CONSTRAINT action_id      IF NOT EXISTS FOR (a:Action)         REQUIRE a.id IS UNIQUE;
CREATE CONSTRAINT anomaly_id     IF NOT EXISTS FOR (a:Anomaly)        REQUIRE a.id IS UNIQUE;
CREATE CONSTRAINT call_id        IF NOT EXISTS FOR (c:Call)           REQUIRE c.id IS UNIQUE;
```

### 1.4 Create Indexes

```cypher
CREATE INDEX app_name          IF NOT EXISTS FOR (a:Application)    ON (a.name);
CREATE INDEX storage_name      IF NOT EXISTS FOR (s:Storage)        ON (s.name);
CREATE INDEX network_name      IF NOT EXISTS FOR (n:Network)        ON (n.name);
CREATE INDEX incident_time     IF NOT EXISTS FOR (i:IncidentTicket) ON (i.startTime, i.endTime);
CREATE INDEX incident_severity IF NOT EXISTS FOR (i:IncidentTicket) ON (i.severity);
CREATE INDEX anomaly_time      IF NOT EXISTS FOR (a:Anomaly)        ON (a.startTime, a.endTime);
CREATE INDEX anomaly_status    IF NOT EXISTS FOR (a:Anomaly)        ON (a.status);
```

### 1.5 Verify the Schema

```cypher
SHOW CONSTRAINTS;   -- expect 9 NODE_PROPERTY_UNIQUENESS constraints
SHOW INDEXES;       -- expect 7 explicit + 2 system LOOKUP indexes
```

All custom indexes should report `state: ONLINE` and `populationPercent: 100.0`. On a fresh database, every node-count query should return zero:

```cypher
MATCH (n:Application)    RETURN count(n) AS applications;
MATCH (n:Storage)        RETURN count(n) AS storage;
MATCH (n:Network)        RETURN count(n) AS networks;
MATCH (n:IncidentTicket) RETURN count(n) AS incidents;
MATCH (n:ChangeTicket)   RETURN count(n) AS changes;
MATCH (n:RCATicket)      RETURN count(n) AS rcas;
MATCH (n:Action)         RETURN count(n) AS actions;
MATCH (n:Anomaly)        RETURN count(n) AS anomalies;
MATCH (n:Call)           RETURN count(n) AS calls;
```

---

## 2. Neo4j Topology Schema

The topology graph models infrastructure as a directed graph: applications, storage, and network resources are nodes; relationships like `CALLS`, `USES_STORAGE`, and `CONNECTS_TO` capture dependencies. On top of this topology, the schema also supports anomaly tracking, incident management, change management, and root cause analysis.

### 2.1 Node Labels

| Label | Purpose | Key Properties |
|-------|---------|----------------|
| `Application` | Apps/services in the topology | `id`, `name`, `tier` (frontend/backend), `owner`, `criticality` |
| `Storage` | Storage resources (DBs, disks) | `id`, `name`, `type` (SQL/NoSQL), `capacity` |
| `Network` | Network elements (switches, routers, VPCs) | `id`, `name`, `type`, `location` |
| `IncidentTicket` | Operational incidents | `id`, `severity`, `status`, `startTime`, `endTime` |
| `ChangeTicket` | Changes impacting topology | `id`, `description`, `startTime`, `endTime`, `status` |
| `RCATicket` | Root cause analysis records | `id`, `description`, `status` |
| `Action` | Corrective actions linked to an RCA | `id`, `description`, `owner`, `status` |
| `Anomaly` | Detected abnormal behaviour | `id`, `type`, `severity`, `status`, `startTime`, `endTime` |
| `Call` *(optional)* | Reified app-to-app interactions | `id`, `latency`, `errorRate` |

### 2.2 Relationships

| Relationship | Direction | Properties | Notes |
|--------------|-----------|------------|-------|
| `CALLS` | Application → Application | `startTime`, `endTime` | Dependency / traversal |
| `USES_STORAGE` | Application → Storage | `startTime`, `endTime` | Layered dependency |
| `CONNECTS_TO` | Application → Network | optional | Connectivity |
| `STORED_ON_NETWORK` | Storage → Network | optional | Layered dependency |
| `IMPACTS` | IncidentTicket → App/Storage/Network | `startTime`, `endTime`, `role` | Time-aware impact |
| `AFFECTS` | ChangeTicket → App/Storage/Network | `startTime`, `endTime`, `description` | Change analysis |
| `ROOT_CAUSE_OF` | RCATicket → IncidentTicket | none | Links RCA to incident |
| `HAS_ACTION` | RCATicket → Action | none | Corrective actions |
| `DETECTED_ON` | Anomaly → App/Storage/Network | `startTime`, `endTime`, `severity` | Used for root cause |
| `TO` | Call → Application | `latency`, `errorRate` | Optional SRE analysis |
| `DEPENDS_ON_TRANSITIVE` | Application → any | none | Precomputed derived edge for blast-radius optimisation |

### 2.3 Traversal Guidelines

- **Root cause analysis** — traverse `CALLS`, `USES_STORAGE`, `CONNECTS_TO` edges forward from impacted nodes, filter for active anomalies, and limit depth.
- **Blast radius / impact** — reverse-traverse edges from a failed node, optionally using precomputed `DEPENDS_ON_TRANSITIVE` edges.
- **Ranking** — prioritise anomalies by severity and confidence.

### 2.4 Sample Data

**Applications:**

```cypher
CREATE (a1:Application {id:'app-123', name:'PaymentsService', tier:'backend', owner:'team-payments', criticality:'high'});
CREATE (a2:Application {id:'app-124', name:'OrdersService',   tier:'backend', owner:'team-orders',   criticality:'medium'});
```

**Storage and Network:**

```cypher
CREATE (s1:Storage {id:'db-001', name:'PaymentsDB', type:'SQL',    capacity:'500GB'});
CREATE (s2:Storage {id:'db-002', name:'OrdersDB',   type:'NoSQL',  capacity:'1TB'});
CREATE (n1:Network {id:'net-01', name:'VPC-East',   type:'VPC',    location:'us-east-1'});
CREATE (n2:Network {id:'net-02', name:'VPC-West',   type:'VPC',    location:'us-west-2'});
```

**Relationships:**

```cypher
MATCH (a1:Application {id:'app-123'}), (a2:Application {id:'app-124'})
CREATE (a1)-[:CALLS]->(a2);

MATCH (a:Application {id:'app-123'}), (s:Storage {id:'db-001'})
CREATE (a)-[:USES_STORAGE]->(s);

MATCH (a:Application {id:'app-123'}), (n:Network {id:'net-01'})
CREATE (a)-[:CONNECTS_TO]->(n);

MATCH (s:Storage {id:'db-001'}), (n:Network {id:'net-01'})
CREATE (s)-[:STORED_ON_NETWORK]->(n);
```

**Anomalies, Incidents, RCA, Actions:**

```cypher
MATCH (a:Application {id:'app-123'})
CREATE (anom:Anomaly {id:'ANOM-1', type:'latency_spike', severity:'high', status:'active', startTime: datetime()})
CREATE (anom)-[:DETECTED_ON]->(a);

MATCH (a:Application {id:'app-123'})
CREATE (inc:IncidentTicket {id:'INC-101', severity:'SEV1', status:'active', startTime: datetime()})
CREATE (inc)-[:IMPACTS]->(a);

CREATE (rca:RCATicket {id:'RCA-1', description:'Investigate latency spike', status:'open'});
CREATE (action:Action {id:'ACT-1', description:'Restart service', owner:'team-payments', status:'pending'});
CREATE (rca)-[:HAS_ACTION]->(action);
MATCH (inc:IncidentTicket {id:'INC-101'}) CREATE (rca)-[:ROOT_CAUSE_OF]->(inc);
```

### 2.5 Optional: Precompute Blast Radius

```cypher
MATCH (a:Application)-[:USES_STORAGE]->(s:Storage)-[:STORED_ON_NETWORK]->(n:Network)
MERGE (a)-[:DEPENDS_ON_TRANSITIVE]->(s)
MERGE (a)-[:DEPENDS_ON_TRANSITIVE]->(n);
```

### 2.6 Example Traversal Queries

**Root cause traversal:**

```cypher
MATCH (start:Application {id: 'app-123'})
MATCH path = (start)-[:CALLS|USES_STORAGE|CONNECTS_TO|STORED_ON_NETWORK*1..5]->(n)
MATCH (a:Anomaly)-[:DETECTED_ON]->(n)
WHERE a.status = 'active'
RETURN n, a, path
LIMIT 50;
```

**Blast radius:**

```cypher
MATCH (failed:Application {id:'app-123'})<-[:DEPENDS_ON_TRANSITIVE]-(impacted)
RETURN impacted;
```

### 2.7 Example JSON Payloads

```json
{
  "id": "app-123",
  "name": "PaymentsService",
  "tier": "backend",
  "owner": "team-payments",
  "criticality": "high"
}
```

```json
{
  "id": "INC-101",
  "severity": "SEV1",
  "status": "resolved",
  "startTime": "2026-03-25T10:00:00",
  "endTime": "2026-03-25T11:00:00"
}
```

```json
{
  "id": "ANOM-1",
  "type": "latency_spike",
  "status": "active",
  "severity": "high",
  "startTime": "2026-03-25T09:55:00",
  "endTime": null
}
```

---

## 3. MySQL

MySQL holds structured session data, LangGraph checkpoints (for replay and resume), the agent memory bank, and the feedback log used by the mitigation feedback loop.

### 3.1 Installation

```bash
brew install mysql
brew services start mysql
mysql_secure_installation
```

### 3.2 Verify Installation

```bash
brew services list | grep mysql
mysql -u root -p
```

Inside the MySQL shell:

```sql
SELECT VERSION();
SHOW DATABASES;
```

### 3.3 Create Database and User

```sql
CREATE DATABASE incident_response
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'ir_user'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON incident_response.* TO 'ir_user'@'localhost';
FLUSH PRIVILEGES;

USE incident_response;
```

### 3.4 Create Tables

```sql
-- Sessions: one row per incident response run
CREATE TABLE sessions (
    id           VARCHAR(36)   NOT NULL,
    created_at   DATETIME      NOT NULL DEFAULT NOW(),
    updated_at   DATETIME      NOT NULL DEFAULT NOW() ON UPDATE NOW(),
    status       ENUM('active','resolved','escalated','stalled') DEFAULT 'active',
    incident_id  VARCHAR(64),
    trigger_node VARCHAR(64),
    severity     VARCHAR(20),
    summary      TEXT,
    PRIMARY KEY (id),
    INDEX idx_status     (status),
    INDEX idx_incident   (incident_id),
    INDEX idx_created_at (created_at)
);

-- LangGraph checkpoints: serialised agent state for replay and fault recovery
CREATE TABLE checkpoints (
    thread_id            VARCHAR(36)   NOT NULL,
    checkpoint_ns        VARCHAR(128)  NOT NULL DEFAULT '',
    checkpoint_id        VARCHAR(36)   NOT NULL,
    parent_checkpoint_id VARCHAR(36),
    type                 VARCHAR(64),
    checkpoint           LONGBLOB      NOT NULL,
    metadata             LONGBLOB,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id),
    INDEX idx_thread (thread_id)
);

-- Agent memory bank: long-term facts accumulated across sessions
CREATE TABLE agent_memory (
    id           BIGINT        NOT NULL AUTO_INCREMENT,
    agent_name   VARCHAR(64)   NOT NULL,
    memory_type  ENUM('fact','pattern','preference','feedback') NOT NULL,
    content      TEXT          NOT NULL,
    context_key  VARCHAR(128),
    confidence   FLOAT         DEFAULT 1.0,
    created_at   DATETIME      DEFAULT NOW(),
    session_id   VARCHAR(36),
    PRIMARY KEY (id),
    INDEX idx_agent       (agent_name),
    INDEX idx_context_key (context_key),
    INDEX idx_session     (session_id)
);

-- Feedback log: records each feedback loop cycle between agents
CREATE TABLE feedback_log (
    id           BIGINT        NOT NULL AUTO_INCREMENT,
    session_id   VARCHAR(36)   NOT NULL,
    from_agent   VARCHAR(64),
    to_agent     VARCHAR(64),
    feedback_msg TEXT,
    iteration    INT           DEFAULT 1,
    created_at   DATETIME      DEFAULT NOW(),
    PRIMARY KEY (id),
    INDEX idx_session (session_id)
);
```

### 3.5 Verify the Schema

```sql
USE incident_response;
SHOW TABLES;
```

Expected:

```
+-----------------------------+
| Tables_in_incident_response |
+-----------------------------+
| agent_memory                |
| checkpoints                 |
| feedback_log                |
| sessions                    |
+-----------------------------+
```

Inspect individual structures:

```sql
DESCRIBE sessions;
DESCRIBE checkpoints;
DESCRIBE agent_memory;
DESCRIBE feedback_log;
```

### 3.6 Verify Python Connectivity

```bash
pip install sqlalchemy pymysql cryptography
```

```python
from sqlalchemy import create_engine, text

engine = create_engine("mysql+pymysql://ir_user:your_password@localhost/incident_response")
with engine.connect() as conn:
    for row in conn.execute(text("SHOW TABLES")):
        print(row[0])
```

### 3.7 Table Usage Summary

| Table | Written by | Read by |
|-------|------------|---------|
| `sessions` | Trigger CLI, Supervisor, Summarizer | All agents (for context) |
| `checkpoints` | LangGraph `MySQLSaver` on every node boundary | LangGraph resume / replay |
| `agent_memory` | Summarizer, individual agents (patterns, facts) | All agents (memory recall) |
| `feedback_log` | Mitigator when confidence is low | Observability / debugging |

---

## 4. Qdrant

### 4.1 Installation

**macOS — direct binary (recommended for local dev):**

```bash
# Apple Silicon
curl -L https://github.com/qdrant/qdrant/releases/latest/download/qdrant-aarch64-apple-darwin.tar.gz | tar xz
chmod +x qdrant && sudo mv qdrant /usr/local/bin/

# Intel Mac
curl -L https://github.com/qdrant/qdrant/releases/latest/download/qdrant-x86_64-apple-darwin.tar.gz | tar xz
chmod +x qdrant && sudo mv qdrant /usr/local/bin/
```

**Docker:**

```bash
docker run -d --name qdrant \
  -p 6333:6333 \
  -v ~/qdrant_storage:/qdrant/storage \
  qdrant/qdrant
```

### 4.2 Start and Verify

```bash
mkdir -p ~/qdrant_data && cd ~/qdrant_data && qdrant           # foreground
nohup qdrant > ~/qdrant_data/qdrant.log 2>&1 &                  # background
curl http://localhost:6333/healthz
```

The dashboard is available at `http://localhost:6333/dashboard`.

### 4.3 Collection Design

Four collections are provisioned at startup, all with 768-dim vectors and cosine similarity.

| Collection | Content | Payload Fields | Used by |
|------------|---------|----------------|---------|
| `mitigation_workflows` | Step-by-step runbooks for fixing incidents | `workflow_id`, `name`, `trigger_type`, `service_type`, `severity`, `steps_json` | Incident Mitigator |
| `rca_documents` | Historical RCA reports and post-mortems | `rca_id`, `incident_id`, `node_id`, `root_cause_summary`, `created_at`, `tags` | Root Cause Finder |
| `incident_summaries` | AI-generated summaries of resolved incidents | `session_id`, `incident_id`, `severity`, `duration_mins`, `resolution`, `created_at` | Summarizer (write), all agents (read) |
| `change_context` | Recent change tickets affecting infrastructure | `change_id`, `node_id`, `description`, `changed_at`, `status` | Root Cause Finder |

### 4.4 Create Collections and Payload Indexes — `setup_qdrant.py`

```python
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance, PayloadSchemaType

client = QdrantClient(host="localhost", port=6333)

collections = {
    "mitigation_workflows": "Step-by-step runbooks for fixing incidents",
    "rca_documents":        "Historical RCA reports and post-mortems",
    "incident_summaries":   "AI-generated summaries of resolved incidents",
    "change_context":       "Recent change tickets affecting infrastructure nodes",
}

existing = [c.name for c in client.get_collections().collections]
for name in collections:
    if name not in existing:
        client.create_collection(
            collection_name=name,
            vectors_config=VectorParams(size=768, distance=Distance.COSINE),
        )
        print(f"  created: {name}")
    else:
        print(f"  skipped (already exists): {name}")

# Payload indexes for fast filtering
client.create_payload_index("mitigation_workflows", "trigger_type", PayloadSchemaType.KEYWORD)
client.create_payload_index("mitigation_workflows", "service_type", PayloadSchemaType.KEYWORD)
client.create_payload_index("mitigation_workflows", "severity",     PayloadSchemaType.KEYWORD)

client.create_payload_index("rca_documents", "node_id",     PayloadSchemaType.KEYWORD)
client.create_payload_index("rca_documents", "incident_id", PayloadSchemaType.KEYWORD)

client.create_payload_index("incident_summaries", "severity",   PayloadSchemaType.KEYWORD)
client.create_payload_index("incident_summaries", "session_id", PayloadSchemaType.KEYWORD)

client.create_payload_index("change_context", "node_id", PayloadSchemaType.KEYWORD)
client.create_payload_index("change_context", "status",  PayloadSchemaType.KEYWORD)

print("\nAll collections and indexes ready.")
```

Run with `python3 setup_qdrant.py` after installing `qdrant-client`.

### 4.5 Verify Qdrant Schema

```python
from qdrant_client import QdrantClient

client = QdrantClient(host="localhost", port=6333)
expected = ["mitigation_workflows", "rca_documents", "incident_summaries", "change_context"]

print("Collections:")
for col in client.get_collections().collections:
    info = client.get_collection(col.name)
    print(f"  {col.name}: {info.points_count} points, vector size {info.config.params.vectors.size}")

print("\nPayload indexes:")
for name in expected:
    info = client.get_collection(name)
    for field, schema in info.payload_schema.items():
        print(f"  {name}.{field}: {schema.data_type}")
```

Expected output shows four collections with zero points and vector size 768, plus all nine payload indexes.

---

## 5. End-to-End Verification Checklist

- [ ] Neo4j responds on `http://localhost:7474` and Bolt `:7687`
- [ ] `SHOW CONSTRAINTS` returns nine uniqueness constraints
- [ ] `SHOW INDEXES` returns all seven custom indexes in `ONLINE` state
- [ ] MySQL `incident_response` database exists with four tables
- [ ] Python can open a connection and list all four tables
- [ ] Qdrant `/healthz` endpoint returns JSON with title and version
- [ ] Four Qdrant collections are created with vector size 768
- [ ] All nine Qdrant payload indexes are present

Once every item is checked, the data layer is ready for the Service Layer (graphserv, MCP servers) and the Agentic Layer (LangGraph).
