# Knowledge Graph

This document describes the triple-store architecture that underpins the knowledge pipeline: why three databases are used, what each stores, how graph relationships are modeled, and how the graph layer integrates with the rest of the system.

## Triple-Store Architecture

The knowledge pipeline distributes data across three purpose-built databases. Each handles a specific class of operation that the others handle poorly or not at all:

```
+-------------------+     +-------------------+     +-------------------+
|   PostgreSQL      |     |     Redis         |     |   OrientDB        |
|   + pgvector      |     |                   |     |                   |
|-------------------|     |-------------------|     |-------------------|
| Structured data   |     | Queue management  |     | Graph traversal   |
| Vector similarity |     | Rate limiting     |     | Relationship      |
| ACID transactions |     | Ephemeral cache   |     |   queries         |
| Full-text search  |     | Pub/sub           |     | Multi-hop paths   |
+-------------------+     +-------------------+     +-------------------+
        |                         |                         |
        |    REST API (aio-01:5000)                         |
        +-------------------------+-------------------------+
```

### Why Three Databases

A single database cannot serve all three roles efficiently:

| Requirement | PostgreSQL | Redis | OrientDB |
|-------------|-----------|-------|----------|
| Store 315K+ documents with metadata | Strong | Unsuitable (memory-bound) | Possible but wasteful |
| Sub-millisecond vector similarity | Strong (pgvector HNSW) | N/A | N/A |
| High-throughput queue drain (4,700 docs/hr) | Possible but heavy | Designed for this | Overkill |
| Multi-hop graph traversal | Recursive CTEs (slow) | N/A | Native O(1) edge traversal |
| Transactional consistency | ACID | Eventual | ACID per transaction |

PostgreSQL excels at structured storage and vector search but requires recursive CTEs for graph-like queries, which become expensive beyond 2-3 hops. Redis excels at ephemeral high-throughput operations but cannot store persistent structured data efficiently. OrientDB provides native graph traversal where following edges between nodes is O(1) regardless of graph size.

## OrientDB Node Types

The OrientDB graph stores four categories of nodes:

| Node Type | Count | Description |
|-----------|-------|-------------|
| Infrastructure | 9 | Fleet nodes (aio-01, server-01/02/03, laptop-01/02, pi-01/02, desktop-ap, server-ap) |
| Workflows | 170+ | Workflow definitions and execution records |
| Models | 200+ | LLM API endpoints (OpenRouter, Anthropic, Google, Groq, etc.) |
| Documents | 315,000+ | Knowledge corpus entries with source metadata |

## Edge Types

Edges encode the relationships between nodes:

| Edge Type | From | To | Meaning |
|-----------|------|-----|---------|
| EXECUTED_ON | Workflow | Infrastructure | Which node ran a workflow |
| USED_MODEL | Workflow | Model | Which LLM was invoked |
| DEPENDS_ON | Workflow | Workflow | Execution dependencies between workflows |
| CONNECTED_TO | Infrastructure | Infrastructure | Network connectivity between fleet nodes |
| JUDGED_BY | Worker | Arbiter | Which arbiter evaluated a worker's output |
| COMPLETED_WITH | Workflow | Arbiter | Final arbiter decision for a workflow |
| RELATED_TO | Document | Document | Semantic similarity above threshold |

## Graph Queries That SQL Cannot Efficiently Express

The primary justification for a dedicated graph database is multi-hop traversal. Consider these queries:

**"Which models have been used on workflows that ran on server-01 and also produced outputs judged by the opus arbiter?"**

In SQL, this requires multiple JOINs across three tables with subqueries:

```sql
SELECT DISTINCT m.name
FROM workflows.executions e
JOIN workflows.worker_results wr ON wr.execution_id = e.id
JOIN workflows.arbiter_decisions ad ON ad.execution_id = e.id
WHERE e.executed_on = 'server-01'
  AND ad.model = 'opus';
```

In OrientDB's graph query language, edges are first-class:

```sql
SELECT expand(out('USED_MODEL'))
FROM (
  SELECT FROM Workflow
  WHERE out('EXECUTED_ON').name = 'server-01'
    AND out('COMPLETED_WITH').model = 'opus'
)
```

For 2-3 hop queries, the SQL and graph approaches are comparable. Beyond that, graph traversal scales linearly with path length while SQL JOINs scale combinatorially.

**Shortest path:** "What is the shortest dependency chain between workflow A and workflow B?"

```sql
SELECT shortestPath(
  (SELECT FROM Workflow WHERE id = 'wf-A'),
  (SELECT FROM Workflow WHERE id = 'wf-B'),
  'OUT', 'DEPENDS_ON'
)
```

This is a single function call in OrientDB. In PostgreSQL, it requires a recursive CTE with cycle detection, which is both harder to write correctly and slower to execute on deep graphs.

## Graph Sync

Workflow completion triggers a graph sync that creates or updates nodes and edges in OrientDB. This sync is **non-blocking** -- graph failures never break the pipeline:

```
Workflow completes
    |
    +---> Store results in PostgreSQL (required, synchronous)
    |
    +---> Sync to OrientDB (optional, async)
              |
              +---> Success: nodes/edges created via MERGE (idempotent)
              |
              +---> Failure: logged, workflow result unaffected
```

The `WorkflowGraphSync` class uses MERGE operations to avoid duplicates. If a Workflow node with the same ID already exists, it is updated rather than duplicated. This makes the sync idempotent -- running it twice for the same workflow produces the same graph state.

If OrientDB is unavailable (container down, network partition), the sync logs a warning and the system continues operating with PostgreSQL only. A PostgreSQL-based fallback using recursive CTEs handles graph queries in degraded mode:

```javascript
const sync = new WorkflowGraphSync();
await sync.connect();

// Automatically uses OrientDB if available, PostgreSQL CTEs if not
const results = await sync.queryWorkflowGraph('bestModelForTask', {
  taskType: 'code-review',
  minExecutions: 5
});
```

## Similarity Links

RELATED_TO edges between Document nodes are computed from vector similarity in PostgreSQL and materialized into the graph. When a new document is embedded, its vector is compared against existing embeddings. Pairs exceeding a configurable cosine similarity threshold (default: 0.85) generate a RELATED_TO edge:

```
Document: "Python asyncio event loops"
    |
    +-- RELATED_TO (0.91) --> "Python async/await patterns"
    |
    +-- RELATED_TO (0.87) --> "asyncio vs threading in Python"
```

These edges enable graph-based exploration: given a document, traverse RELATED_TO edges to find semantically adjacent content without re-running vector search. This is particularly useful for building context windows -- collect a document and its 2-hop RELATED_TO neighbors to assemble a rich context for LLM prompts.

## REST API

Graph queries are exposed through the controller's REST API:

```
POST /graph/query
Content-Type: application/json

{
  "queryType": "bestModelForTask",
  "params": {
    "taskType": "code-review",
    "minExecutions": 5
  }
}
```

Available query types:

| Query Type | Purpose |
|-----------|---------|
| `workflowsByModel` | Find workflows that used a specific model |
| `bestModelForTask` | Rank models by average quality score for a task type |
| `executionPath` | Trace the worker-to-arbiter execution path of a workflow |
| `modelCollaboration` | Find model pairs that frequently co-occur in workflows |

## Why OrientDB Over Neo4j

The system originally used Neo4j. It was replaced with OrientDB for two reasons:

1. **Persistence bugs in Neo4j Docker deployment:** Data loss occurred across container restarts despite volume mounts being correctly configured. The root cause was not fully diagnosed, but the symptom was reproducible.

2. **Native REST API:** OrientDB exposes a built-in HTTP/REST interface, eliminating the need for a Bolt driver dependency. This simplifies deployment on fleet workers that may not have the Neo4j driver installed and allows graph queries from any HTTP client (curl, Python requests, Node fetch).

The `WorkflowGraphSync` class retains PostgreSQL recursive CTE fallback queries for all graph operations, ensuring that the system functions fully even if OrientDB is offline. OrientDB is a performance optimization for graph traversal, not a hard dependency.
