# Scaling Guide

This guide explains how to scale the distributed LLM orchestration framework horizontally and vertically, and how to identify and resolve pipeline bottlenecks.

---

## Horizontal Scaling

The system is designed so that most components can be scaled by adding more instances without reconfiguring the orchestrator.

### Adding Scrapers

Scrapers are the simplest component to scale. Each scraper subclasses `BaseScraper` and produces documents that feed into the ingestion pipeline.

1. Implement a new scraper by subclassing `BaseScraper`.
2. Deploy it to any fleet worker via `scp`.
3. Launch it. The pipeline handles chunking, embedding, and graph insertion downstream.

No orchestrator changes are needed. The scraper pushes documents into the ingestion queue, and existing pipeline workers consume them.

### Adding Models

New free-tier models that appear on providers (OpenRouter, Groq, Cerebras, DeepSeek) are automatically available to the routing layer. The Thompson Sampling strategy discovers and evaluates new models:

- New models start with a uniform prior (alpha=1, beta=1).
- Early tasks are exploratory -- the sampler tries the new model on low-risk work.
- Performance data accumulates in `learning.strategy_performance`.
- Models that perform well receive increasing traffic share.

No code changes are required to incorporate a new model. The routing layer discovers it from the provider's model list.

### Adding Fleet Nodes

A new worker joins the fleet in three steps:

```bash
# 1. Set up SSH key
ssh-copy-id claude@<new-worker>

# 2. Register with the orchestrator
curl -X POST http://aio-01:5000/fleet/register \
  -H "Content-Type: application/json" \
  -d '{"host": "<new-worker>", "capabilities": ["scrape", "chunk", "workflow"]}'

# 3. Verify
curl http://aio-01:5000/fleet/status
```

Zero orchestrator configuration changes. The circuit breaker system automatically manages the new worker's health state.

### Adding GA Optimizers

Genetic algorithm optimizers follow a consistent pattern:

1. Define a chromosome structure (the parameters being optimized).
2. Define a fitness function (how to evaluate a chromosome).
3. Connect to the REST API for persistence and retrieval.
4. The GA engine (`learning/ga_engine.py`) handles crossover, mutation, and selection.

Existing optimizers to reference:

| Optimizer                         | What It Optimizes                |
|-----------------------------------|----------------------------------|
| `genetic_model_optimizer.py`      | Model routing weights            |
| `ga_rag_retrieval_optimizer.py`   | RAG retrieval parameters         |
| `ga_team_selection_fixed.py`      | Team composition for tasks       |
| `ga_training_data_curator.py`     | Training data selection          |

### Adding Review Layers

The multi-AI review pipeline supports pluggable external reviewers. External models (Grok, ChatGPT, etc.) slot into the adversarial layer:

- Review panel and meta-review panel must have zero model overlap.
- External reviewers are called via their respective APIs.
- Each phase uses a different arbiter to prevent self-confirmation.

See the "Review-Fix Cycle with Meta-Review" pattern in `CLAUDE.md` for panel composition rules.

---

## Vertical Scaling

### PostgreSQL

| Concern              | Approach                                                |
|----------------------|---------------------------------------------------------|
| Connection exhaustion | Use connection pooling (PgBouncer or application-level) |
| Slow vector queries  | Tune `ivfflat` index parameters, increase `lists`       |
| Materialized views   | Adjust refresh interval (default: 5 minutes)            |
| Storage growth       | 90-day retention policy; archive older data              |

### Redis

| Concern              | Approach                                                |
|----------------------|---------------------------------------------------------|
| Memory pressure      | Monitor sorted set sizes; trim completed items          |
| Queue depth          | Scale consumers, not Redis itself                       |
| Persistence          | RDB snapshots are sufficient for queue data             |

### Embedding Throughput

Embedding is typically the pipeline's compute bottleneck. Scale by adding dedicated compute nodes (laptop-class machines with sufficient RAM). Never run embedding workers on fleet workers -- they lack the resources and will degrade other operations.

---

## Pipeline Bottleneck Identification

The ingestion pipeline has four stages. Each has a different scaling profile:

```
Scraping --> Chunking --> Embedding --> Graph Insertion
```

### Stage Characteristics

| Stage     | Bound By | Scaling Strategy                | Indicator of Bottleneck              |
|-----------|----------|---------------------------------|--------------------------------------|
| Scraping  | Network  | Add more scraper workers        | Ingestion queue is empty/low         |
| Chunking  | CPU      | Add more chunk workers          | Chunk queue grows while embed is idle|
| Embedding | Compute  | Add dedicated embedding nodes   | Embed queue grows continuously       |
| Graph     | I/O      | Rarely the bottleneck           | Graph queue grows (unusual)          |

### Diagnosing Bottlenecks

```bash
# Check queue depths across all stages
curl http://aio-01:5000/queue/status
```

Interpretation:

- **Ingestion queue empty, downstream queues idle:** Scraping is too slow. Add scrapers.
- **Chunk queue growing, embed queue empty:** Chunking cannot keep up. Add chunk workers.
- **Embed queue growing continuously:** Embedding is the bottleneck. Add dedicated compute nodes.
- **Graph queue growing:** Unusual. Check OrientDB health and I/O capacity.

---

## Current Scale

As a reference point for capacity planning:

| Metric        | Count     |
|---------------|-----------|
| Documents     | 315,000+  |
| Chunks        | 705,000+  |
| Embeddings    | 705,000+  |
| DB Tables     | 226       |
| Fleet Nodes   | 9         |
| API Models    | 200+      |
| Scrapers      | 60+       |

---

## Capacity Planning Considerations

- **Worker memory:** Nodes with 1GB RAM (pi-01, pi-02, desktop-ap, server-ap) are limited to lightweight tasks (scraping, queue consumption). Heavy processing requires nodes with 8GB+ RAM.
- **Maximum parallel workers:** The orchestrator defaults to 3 concurrent workers per node. Adjust based on node capacity.
- **Database connections:** PostgreSQL's default `max_connections` may need increasing as workers scale. Connection pooling is preferred over raising the limit.
- **NFS dependency:** The fleet requires NFS for shared file access. If NFS is unavailable, workers fall back to local storage, but cross-node file references will fail.
