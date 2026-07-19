# OrientDB Graph Database

OrientDB provides the graph layer for relationships that are natural to
express as nodes and edges but expensive to query with relational self-joins.
It runs as a Docker container on the controller node and is accessed through
the REST API gateway.

**Version:** OrientDB 3.2 (Community Edition)
**Location:** Controller node (aio-01), Docker container
**Ports:** 2424 (binary), 2480 (HTTP)
**Database name:** `orchestration`
**Previous system:** Neo4j (replaced due to persistent data loss in Docker volumes)

---

## Data Model

### Node Types

| Vertex Class    | Count   | Key Properties                              |
|-----------------|---------|---------------------------------------------|
| Infrastructure  | 9       | hostname, ip, role, cpu_cores, memory_gb    |
| Workflow        | 170+    | name, pattern, phases, avg_duration_ms      |
| Model           | 200+    | provider, model_id, cost_per_1k, max_tokens|
| Document        | 315K+   | url, title, source, chunk_count, created_at |
| Concept         | 48K+    | name, type, frequency                       |

### Edge Types

| Edge Class      | From            | To              | Properties              |
|-----------------|-----------------|-----------------|--------------------------|
| EXECUTED_ON     | Workflow        | Infrastructure  | count, last_run, avg_ms  |
| USED_MODEL      | Workflow        | Model           | count, avg_quality       |
| DEPENDS_ON      | Workflow        | Workflow        | dependency_type          |
| CONNECTED_TO    | Infrastructure  | Infrastructure  | latency_ms, bandwidth    |
| REFERENCES      | Document        | Document        | relationship_type        |
| MENTIONS        | Document        | Concept         | count, positions         |
| RELATED_TO      | Concept         | Concept         | strength, co_occurrence  |

### Schema Diagram

```
                    DEPENDS_ON
               +------------------+
               |                  |
               v                  |
  +----------+    EXECUTED_ON    +--------------+
  | Workflow |------------------>| Infrastructure|
  +----------+                   +--------------+
       |                              |
       | USED_MODEL            CONNECTED_TO
       v                              |
  +----------+                        v
  |  Model   |                +--------------+
  +----------+                | Infrastructure|
                              +--------------+

  +----------+   REFERENCES   +----------+
  | Document |--------------->| Document |
  +----------+                +----------+
       |
       | MENTIONS
       v
  +----------+  RELATED_TO   +----------+
  | Concept  |-------------->| Concept  |
  +----------+               +----------+
```

---

## Query Patterns

### Multi-Hop Traversal

Find all documents that were processed by workflows using a specific model
family. This requires traversing three edge types across four vertex classes:

```sql
SELECT document.title, document.url
FROM (
  TRAVERSE out('USED_MODEL'), in('EXECUTED_ON'), out('PROCESSED')
  FROM (SELECT FROM Model WHERE provider = 'anthropic')
  MAXDEPTH 3
)
WHERE @class = 'Document'
```

The equivalent SQL query in PostgreSQL would require:

```sql
SELECT d.title, d.url
FROM models m
JOIN workflow_models wm ON wm.model_id = m.id
JOIN workflows w ON w.id = wm.workflow_id
JOIN workflow_executions we ON we.workflow_id = w.id
JOIN execution_documents ed ON ed.execution_id = we.id
JOIN documents d ON d.id = ed.document_id
WHERE m.provider = 'anthropic';
```

Both return the same result. At depth 3, the performance difference is modest.
At depth 5+, the relational query becomes impractical.

### Shortest Path

Find the shortest relationship path between two infrastructure nodes:

```sql
SELECT shortestPath(
  (SELECT FROM Infrastructure WHERE hostname = 'server-01'),
  (SELECT FROM Infrastructure WHERE hostname = 'pi-02'),
  'BOTH',
  'CONNECTED_TO'
)
```

### Neighborhood Discovery

Find all models and infrastructure nodes within 2 hops of a specific workflow:

```sql
SELECT @class, name, @rid
FROM (
  TRAVERSE both()
  FROM (SELECT FROM Workflow WHERE name = 'deep-research')
  MAXDEPTH 2
)
WHERE @class IN ['Model', 'Infrastructure']
```

### Concept Co-occurrence

Find concepts that frequently appear together across documents:

```sql
SELECT c1.name AS concept_a, c2.name AS concept_b, r.co_occurrence
FROM (
  SELECT expand(out('RELATED_TO'))
  FROM Concept
  WHERE name = 'firmware'
) AS r
LET c1 = $parent.current, c2 = $current
ORDER BY r.co_occurrence DESC
LIMIT 20
```

---

## REST API Access

All graph queries flow through the orchestrator's REST API gateway. No
component connects to OrientDB directly.

### Query Endpoint

```
POST /graph/query
Content-Type: application/json

{
  "query": "SELECT FROM Infrastructure WHERE role = 'worker'",
  "params": {}
}
```

### Traversal Endpoint

```
POST /graph/traverse
Content-Type: application/json

{
  "start_class": "Workflow",
  "start_filter": {"name": "deep-research"},
  "edge_types": ["USED_MODEL", "EXECUTED_ON"],
  "max_depth": 3,
  "target_class": "Infrastructure"
}
```

The traversal endpoint provides a structured interface that generates the
underlying OrientDB SQL, handling parameter escaping and result pagination.

---

## Why Not PostgreSQL for Graphs

PostgreSQL can represent graphs using adjacency tables (foreign keys between
rows). For shallow queries (1-2 hops), this works well. The cost grows
non-linearly with depth.

```
+-------------------+------------------+------------------------+
| Query Depth       | PostgreSQL       | OrientDB               |
+-------------------+------------------+------------------------+
| 1 hop             | 1 JOIN           | 1 TRAVERSE step        |
|                   | ~0.5 ms          | ~0.8 ms                |
+-------------------+------------------+------------------------+
| 2 hops            | 2 JOINs          | 2 TRAVERSE steps       |
|                   | ~1.2 ms          | ~1.1 ms                |
+-------------------+------------------+------------------------+
| 3 hops            | 4 JOINs          | 3 TRAVERSE steps       |
|                   | ~4.8 ms          | ~1.5 ms                |
+-------------------+------------------+------------------------+
| 5 hops            | 10+ JOINs       | 5 TRAVERSE steps       |
|                   | ~80+ ms          | ~2.5 ms                |
+-------------------+------------------+------------------------+
| Variable depth    | Recursive CTE    | MAXDEPTH parameter     |
|                   | (hard to optimize)| (index-free adjacency) |
+-------------------+------------------+------------------------+
```

OrientDB uses index-free adjacency: each vertex holds direct pointers to its
edges, making traversal cost proportional to the result set size rather than
the total graph size. PostgreSQL must scan indices at each join level,
multiplying cost with depth.

The system uses both databases for their strengths: PostgreSQL for structured
data, aggregation, and vector search; OrientDB for relationship traversal and
pattern discovery.

---

## Resilience

Graph synchronization is non-blocking. If OrientDB is unavailable:

- The ingestion pipeline continues without graph updates
- Missed updates are queued in Redis and replayed when the connection recovers
- No workflow fails due to a graph write failure
- Read queries against the graph return empty results with a warning flag,
  and callers fall back to PostgreSQL-based alternatives

This design ensures that the graph database enhances the system's capabilities
without becoming a single point of failure.

---

## Why OrientDB Over Neo4j

The original deployment used Neo4j Community Edition in Docker. Two issues
motivated the switch:

1. **Data persistence bugs:** Neo4j's Docker volumes intermittently lost data
   during container restarts. The root cause was a combination of filesystem
   flush timing and Neo4j's transaction log rotation. Multiple configuration
   attempts did not resolve it.

2. **Licensing:** Neo4j Community Edition lacks clustering, hot backups, and
   several query optimizations available only in the Enterprise Edition (which
   requires a commercial license for production use).

OrientDB Community Edition provides multi-model (document + graph + key-value)
capabilities, straightforward Docker volume persistence, and an Apache 2.0
license with no feature restrictions.
