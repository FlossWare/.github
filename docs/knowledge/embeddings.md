# Vector Embeddings

This document describes how text chunks are converted into vector embeddings, how those embeddings are indexed for fast retrieval, the compute isolation architecture, and the fallback and caching mechanisms that ensure reliability.

## What Embeddings Are

An embedding is a fixed-length vector of floating-point numbers that encodes the semantic meaning of a text passage. The embedding model maps text into a high-dimensional space where semantically similar passages occupy nearby regions. This is not keyword matching -- it captures meaning:

```
"authentication failure"  -->  [0.12, -0.34, 0.56, ...]  --+
                                                             |  cosine similarity: 0.91
"login error"             -->  [0.11, -0.32, 0.58, ...]  --+

"chocolate cake recipe"   -->  [-0.45, 0.78, -0.12, ...]     cosine similarity: 0.03
```

A query for "authentication failure" will retrieve documents about "login errors", "credential validation issues", and "session token expiration" -- none of which share keywords with the query, but all of which are semantically proximate in the embedding space.

## Embedding Model

The system uses `sentence-transformers/all-mpnet-base-v2`, a pre-trained transformer model that produces 768-dimensional embeddings:

| Property | Value |
|----------|-------|
| Model | all-mpnet-base-v2 |
| Dimensions | 768 |
| Max input tokens | 384 (longer inputs are truncated to 8,000 characters before encoding) |
| Training corpus | 1B+ sentence pairs |
| Similarity metric | Cosine similarity |

This model was selected for its balance of quality and speed. It consistently outperforms smaller models (MiniLM, distilbert) on semantic textual similarity benchmarks while remaining fast enough for batch processing.

## Multiple Embedding Dimensions

The system uses different embedding dimensions for different data types, reflecting the varying semantic complexity of each:

| Data Type | Dimensions | Table | Rationale |
|-----------|-----------|-------|-----------|
| Experiences | 128 | learning.experiences | Low-dimensional; captures task type and outcome |
| Workflows | 384 | workflow.executions | Medium-dimensional; captures task description semantics |
| Knowledge chunks | 768 | knowledge.embeddings | Full-dimensional; maximum semantic fidelity for retrieval |

Lower-dimensional embeddings are faster to compare and index but sacrifice semantic nuance. For knowledge retrieval -- where distinguishing between "Python asyncio event loop" and "Python asyncio coroutine" matters -- full 768-dimensional embeddings are essential.

## HNSW Indexing

Raw vector similarity search is O(n): comparing a query vector against every stored vector. With 315,000+ documents producing millions of chunks, this becomes impractical.

The system uses HNSW (Hierarchical Navigable Small World) indexes via the `pgvector` PostgreSQL extension. HNSW constructs a multi-layer proximity graph that enables approximate nearest-neighbor search in O(log n) time:

```
Layer 2 (sparse):    A -------- D -------- G
                     |                     |
Layer 1 (medium):    A --- C --- D --- F --- G
                     |    |     |     |     |
Layer 0 (dense):     A-B-C-D-E-F-G-H-I-J-K-L
```

Search begins at the top layer (few nodes, long-range links) and descends through progressively denser layers, narrowing the candidate set at each level. The result is approximate but highly accurate -- typical recall exceeds 95% at a fraction of the computational cost of exhaustive search.

### Performance

| Operation | ChromaDB | PostgreSQL + pgvector |
|-----------|----------|----------------------|
| Simple similarity (128-dim) | 0.9ms | 0.44ms |
| Filtered similarity | 2.3ms | 0.44ms |
| 768-dim vectors | 1.2ms | 0.97ms |
| Average query time | -- | 0.4ms |

PostgreSQL with pgvector delivers 2-6x faster query times than ChromaDB, with the additional advantage of transactional consistency, SQL-based filtering, and co-location with the document and chunk data.

## Compute Isolation

Embedding generation is a hard architectural constraint: it runs ONLY on dedicated compute nodes (laptop-01, laptop-02), never on fleet workers (server-01/02/03, pi-01/02, desktop-ap, server-ap).

```
Fleet Workers (server-01, server-02, ...)       Compute Nodes (laptop-01, laptop-02)
+----------------------------------+            +----------------------------------+
| Scraping, store workers,         |            | sentence-transformers model      |
| orchestration tasks              |            | embed-worker.py processes        |
|                                  |            | GPU/CPU inference                |
| NO sentence-transformers         |            |                                  |
| NO embedding generation          |            | Dedicated to ML workloads        |
+----------------------------------+            +----------------------------------+
                                                         |
                                                POST /storage/query
                                                POST /learning/embeddings/generate
                                                         |
                                                         v
                                               +------------------+
                                               | PostgreSQL       |
                                               | knowledge.       |
                                               | embeddings       |
                                               +------------------+
```

**Why this separation matters:** The `all-mpnet-base-v2` model requires significant memory (400MB+ loaded) and CPU time per inference batch. Running it on fleet workers (which have as little as 1GB total RAM on pi nodes) would destabilize those nodes. Concentrating ML inference on the laptops (31GB RAM, multi-core CPUs) keeps the fleet stable for orchestration while dedicating appropriate hardware to compute-intensive work.

## Embedding Workers

Embedding workers (`scripts/embed-worker.py`) directly query PostgreSQL for chunks that lack embeddings, generate vectors locally, and store the results:

```python
# Claim a range of chunk IDs to avoid conflicts between workers
python3 embed-worker.py 0 30000       # worker 1: chunks 0-30000
python3 embed-worker.py 30000 30000   # worker 2: chunks 30000-60000
```

Each worker:

1. Queries `knowledge.chunks` for IDs in its assigned range that have no corresponding row in `knowledge.embeddings`.
2. Batches those chunks (default batch size: 50) and truncates each to 8,000 characters.
3. Runs the sentence-transformers model locally to produce 768-dim vectors.
4. Inserts results into `knowledge.embeddings` with `ON CONFLICT` upsert semantics.
5. Reports progress every 30 seconds.

Range-based partitioning eliminates coordination overhead. Two workers processing non-overlapping ID ranges will never conflict. The `ON CONFLICT` clause ensures that if ranges do overlap (e.g., during re-embedding), the latest embedding wins.

## Cascading Fallback

If local embedding generation fails (model not loaded, out of memory, library not installed), the worker falls back to the controller's REST API:

```
embed_batch(texts)
    |
    +---> Try local sentence-transformers
    |         |
    |         +---> Success: return vectors (provider="local")
    |         |
    |         +---> Failure: set _local_failed = True
    |
    +---> Fall back to POST /learning/embeddings/generate
              |
              +---> Success: return vectors (provider="api")
              |
              +---> Failure: return None (chunk skipped, retried later)
```

The `_local_failed` flag is sticky for the lifetime of the process -- once local inference fails, subsequent batches go straight to the API endpoint without retrying local. This avoids repeated slow failures (e.g., OOM on every batch).

## Embedding Pool

The `EmbeddingPool` class (`shared/embedding-pool.cjs`) throttles concurrent embedding generation to prevent CPU overload on the compute nodes. It maintains a queue of pending requests and limits concurrent processes to a configurable maximum (default: 4):

```javascript
const { getEmbeddingPool } = require('./shared/embedding-pool.cjs');
const pool = getEmbeddingPool({ maxConcurrent: 4, timeout: 120000 });

const embedding = await pool.generate("text to embed");
const embeddings = await pool.generate(["batch", "of", "texts"]);

console.log(pool.getStatus());
// { running: 2, queued: 5, maxConcurrent: 4 }
```

When all slots are occupied, new requests queue until a slot opens. Each embedding subprocess has a 120-second timeout to handle model cold-start (first inference loads the model into memory).

## LRU Cache

A 1,000-entry LRU cache prevents redundant re-embedding of identical text. Cache keys are deterministic SHA-256 hashes of the input text, so the same text always produces a cache hit regardless of request origin.

When the cache is full and the embedding endpoint is unavailable, a hash-based fallback generates a deterministic pseudo-embedding from the text hash. These fallback embeddings have no semantic meaning but provide a stable vector for storage, allowing the pipeline to continue. They are tagged with `provider="fallback"` and re-embedded with the real model during the next successful batch run.

## Storage Schema

Embeddings are stored in the `knowledge.embeddings` table with pgvector's `vector(768)` type:

```sql
CREATE TABLE knowledge.embeddings (
    id         SERIAL PRIMARY KEY,
    chunk_id   INTEGER NOT NULL REFERENCES knowledge.chunks(id) ON DELETE CASCADE,
    provider   TEXT NOT NULL,              -- "local", "api", or "fallback"
    model      TEXT NOT NULL,              -- "all-mpnet-base-v2"
    embedding  vector(768),               -- pgvector column
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(chunk_id, provider)
);
```

The `UNIQUE(chunk_id, provider)` constraint prevents duplicate embeddings per chunk per provider. The `ON DELETE CASCADE` foreign key ensures that deleting a chunk automatically removes its embeddings.

Vector similarity queries use pgvector's cosine distance operator:

```sql
SELECT c.content, c.document_id,
       e.embedding <=> query_vector AS distance
FROM knowledge.embeddings e
JOIN knowledge.chunks c ON c.id = e.chunk_id
ORDER BY e.embedding <=> query_vector
LIMIT 10;
```

The `<=>` operator computes cosine distance (1 - cosine similarity), so lower values indicate higher similarity. Combined with the HNSW index, this query executes in under 1ms for typical corpus sizes.
