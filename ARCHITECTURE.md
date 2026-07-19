# FlossWare Architecture Guide

**Version:** 1.0  
**Status:** Living document  
**Audience:** Senior/principal/distinguished engineers, architects, AI researchers  
**Scope:** Distributed LLM orchestration framework -- design, internals, operational model  
**License:** Apache 2.0

---

**Table of Contents (Part 1)**

1. [Vision and Scope](#1-vision-and-scope)
2. [Design Philosophy](#2-design-philosophy)
3. [Architectural Principles](#3-architectural-principles)
4. [Overall System Architecture](#4-overall-system-architecture)
5. [Distributed Fleet](#5-distributed-fleet)
6. [Orchestration Layer](#6-orchestration-layer)
7. [Consensus Engine](#7-consensus-engine)
8. [Multi-AI Review Pipeline](#8-multi-ai-review-pipeline)

---

## 1. Vision and Scope

### The Problem

A single large language model, no matter how capable, has a structural weakness: it cannot adversarially challenge its own outputs. When GPT-4o reviews code, it produces findings with high confidence. When you ask it to verify those findings, it largely agrees with itself. This is not a bug -- it is an inherent property of autoregressive generation. The model's posterior distribution over tokens does not meaningfully shift when asked to "double-check."

The consequence is a false-positive rate that compounds with scale. In production code review across repositories with 50,000+ lines of code, single-model review pipelines routinely produce 30-60% false positives. Each false positive consumes developer attention, erodes trust in automated tooling, and eventually leads teams to ignore automated findings entirely -- including the true positives buried among them.

### The Thesis

This system is built on a single architectural bet: **independent adversarial review across heterogeneous models produces higher-quality outputs than any single model operating alone**, regardless of that model's capability level.

The key word is *independent*. Two instances of the same model are not independent -- they share weights, training data, and systematic biases. Claude Opus reviewing code and then Claude Opus meta-reviewing those findings is circular validation. But Claude Opus reviewing code and then Hermes-405B, Nemotron-Ultra-550B, and Qwen3-Next-80B adversarially challenging those findings introduces genuine epistemic diversity. These models were trained by different organizations, on different data mixtures, with different optimization objectives. Their failure modes are uncorrelated.

### What This System Is

This is a **distributed orchestration framework** over pre-trained large language models. It routes tasks to appropriate models, collects independent responses, synthesizes consensus, and evolves its routing strategies based on observed outcomes. The models themselves are unchanged -- no fine-tuning, no weight updates, no training of any kind occurs within this system.

All improvements in system behavior are attributable to three mechanisms:

1. **Routing efficiency** -- Matching tasks to models with demonstrated strength in that task category.
2. **Task decomposition** -- Breaking complex work into parallelizable subtasks distributed across a heterogeneous fleet.
3. **Iterative review loops** -- Fix-then-verify cycles with panel rotation that converge on correctness through adversarial pressure.

### What This System Is Not

This system does not train models. It does not perform gradient descent. It does not update weights. It does not achieve emergent intelligence. It does not approach AGI. These are important boundaries to state explicitly because orchestration systems that use terms like "learning" and "evolution" can easily be misread as claiming capabilities they do not possess.

When this document says the system "learns," it means: the routing layer updates probability distributions over model-task assignments based on observed success rates (Thompson Sampling). When it says the system "evolves," it means: genetic algorithms mutate and crossover configuration parameters like team composition, consensus thresholds, and retry strategies. The underlying models remain exactly as their creators shipped them.

### Scope Boundaries

| In Scope | Out of Scope |
|----------|-------------|
| Multi-model orchestration and consensus | Model training or fine-tuning |
| Task classification and intelligent routing | Prompt engineering research |
| Distributed fleet execution over SSH | Container orchestration (Kubernetes) |
| Adversarial review pipelines | Single-model inference optimization |
| Genetic algorithm parameter evolution | Neural architecture search |
| Provider-agnostic model access | Vendor-specific SDK wrappers |

---

## 2. Design Philosophy

Six principles govern every design decision in this system. They are listed in priority order -- when two principles conflict, the higher-numbered principle yields.

### Orchestration Over Training

The system treats models as opaque functions: prompt in, completion out. It never attempts to modify model internals. This is a deliberate choice, not a limitation.

The alternative -- fine-tuning models for specific tasks -- offers marginal quality improvements at substantial cost: GPU infrastructure, training data curation, evaluation pipelines, model versioning, and the ongoing maintenance burden of custom model deployments. For a system accessing 200+ models via API, the orchestration approach provides broader coverage at lower operational complexity. A well-routed query to the right model outperforms a poorly-routed query to a fine-tuned model.

**Tradeoff acknowledged:** Fine-tuned models can outperform general-purpose models on narrow, well-defined tasks. If the system identifies such a task category (e.g., SQL generation for a specific schema), the architecture supports adding a fine-tuned model as another provider without changing the orchestration layer.

### Independence Over Agreement

The system is designed to maximize epistemic independence between models that evaluate the same artifact. This manifests in several concrete rules:

- Review and meta-review panels share zero models.
- Each phase of a review pipeline uses a different arbiter.
- Models from the same provider family (e.g., Claude Opus and Claude Sonnet) are distributed across phases rather than concentrated in one.
- When a consensus vote is unanimous, the system flags it for scrutiny rather than treating it as high confidence.

The underlying insight is from ensemble learning theory: the error rate of a majority-vote ensemble decreases exponentially with the number of independent classifiers, but only if their errors are uncorrelated. Correlated errors -- which arise from shared training data, shared architectures, or shared systematic biases -- provide no ensemble benefit.

**Tradeoff acknowledged:** Independence comes at the cost of latency and token spend. Querying four models instead of one is 4x the API cost and bounded by the slowest responder. The system mitigates this through parallelism (all panel members execute concurrently) and caching (identical consensus queries within a TTL window return cached results).

### Evolution Over Configuration

System parameters are not manually tuned. They evolve through genetic algorithms that mutate, crossover, and select configurations based on observed fitness metrics.

Evolved parameters include:

- **Team composition:** Which models serve on review vs. meta-review panels.
- **Consensus thresholds:** The minimum agreement level required to accept a finding.
- **Retry budgets:** How many fix-review cycles to attempt before escalating.
- **Routing weights:** Per-model, per-task-category capability scores.
- **RAG retrieval parameters:** Chunk sizes, overlap ratios, similarity thresholds.

The genetic algorithm operates on a population of configuration genomes, each encoding a complete set of these parameters. Fitness is measured by downstream outcomes: false-positive rates, fix success rates, verification pass rates, and cost-per-resolved-issue. The population evolves across generations, with tournament selection, single-point crossover, and Gaussian mutation.

**Tradeoff acknowledged:** Evolutionary optimization is slow. The GA requires hundreds of evaluations to converge, and each evaluation involves running a complete review pipeline. The system amortizes this cost by running evolution in the background and applying the current best genome to production traffic while the next generation is being evaluated.

**Alternative considered:** Bayesian optimization (e.g., Optuna, Hyperopt) would converge faster on continuous parameters but handles the combinatorial structure of team composition poorly. The GA's ability to represent discrete choices (which models, which strategies) as genes that undergo crossover makes it a better fit for this configuration space.

### Interfaces Over Implementations

Every major subsystem is accessed through a defined interface, typically a REST API or a JavaScript/Python adapter module. Workers never import database drivers. Workflows never construct SSH commands. The orchestration layer never parses model-specific response formats.

This principle enables:

- **Provider substitution:** Swapping OpenRouter for a direct API call requires changing one adapter, not every workflow.
- **Database migration:** Moving from PostgreSQL-backed queues to Redis-backed queues required changing one blueprint, not every consumer.
- **Testing:** Subsystems can be tested in isolation by mocking their dependencies at the interface boundary.

**Tradeoff acknowledged:** Interface boundaries add indirection. A direct `psycopg2` call is faster than an HTTP request to a REST API that wraps a `psycopg2` call. The system accepts this latency overhead (typically 1-5ms per hop on a local network) in exchange for the operational benefits of loose coupling.

### Provider Independence

The system is designed to survive the loss of any single model provider. If OpenRouter goes down, traffic routes to direct Anthropic, Google, and Groq APIs. If Anthropic raises prices, the routing layer shifts traffic to comparable models on other providers. No workflow, no consensus strategy, and no review pipeline is hardcoded to a single provider.

This is achieved through a model abstraction layer that normalizes provider-specific request/response formats into a common interface. The routing layer operates on model capabilities and task types, not on provider names.

**Tradeoff acknowledged:** Provider-specific features (Anthropic's prompt caching, Google's context windows, Groq's LPU speed) are underutilized when accessed through a common abstraction. The system addresses this by allowing provider-specific optimizations in the adapter layer while keeping the orchestration layer provider-agnostic.

### Honest Accounting

The system maintains rigorous separation between what it measures and what it claims. Internal metrics (consensus rates, routing efficiency, cost-per-query) are tracked but never cited as evidence of model improvement. The models are not improving -- the orchestration is improving.

This principle extends to documentation. Components are labeled "UNVALIDATED" until independently verified. Metrics include confidence intervals. Claims about system behavior cite specific measurements rather than aspirational descriptions.

**Deep dive:** See `docs/philosophy.md` for extended discussion of these principles, including worked examples of how each principle influenced specific design decisions.

---

## 3. Architectural Principles

The design philosophy above is realized through six concrete architectural rules. Violations of these rules are treated as bugs.

### 3.1 All Access Through the REST API Gateway

Every external interaction with the system passes through the REST API on the controller node (aio-01). Workers do not expose their own APIs. Clients do not SSH into workers directly. Database queries do not originate from worker nodes.

```
  CORRECT                           INCORRECT

  Client                            Client
    |                                 |
    v                                 v
  REST API (aio-01:5000)            Worker (server-02)
    |                                 |
    v                                 v
  Worker (server-02)                PostgreSQL (aio-01:5433)
    |
    v
  REST API (aio-01:5000)
    |
    v
  PostgreSQL (aio-01:5433)
```

**Why:** Centralized access enables rate limiting, authentication, audit logging, circuit breaking, and request routing. Without it, every worker becomes a potential point of uncontrolled database access, and capacity planning becomes impossible.

**Exception:** Health check endpoints on worker nodes (port 7340) are queried directly by the controller for fleet status. These are read-only, stateless, and return a bounded response (max 10KB).

### 3.2 Workers Never Touch Databases Directly

Workers receive tasks via SSH, execute them using API calls to model providers, and return results to the controller. They do not import `psycopg2`, they do not connect to Redis, and they do not write to OrientDB. All persistence is mediated by the controller's REST API.

**Why:** Workers run on heterogeneous hardware (x86_64 servers, ARM64 single-board computers) with varying capabilities. Requiring database drivers on every worker increases the deployment surface, creates version skew risks, and makes it impossible to reason about database connection pool sizes. With the controller as the sole database client, connection pooling is centralized and predictable.

**Implementation:** The fleet executor (`shared/fleet_executor.py`) uses only Python standard library modules -- no third-party dependencies. This ensures it runs on every architecture in the fleet without dependency installation.

### 3.3 Embedding Runs Only on Dedicated Compute Nodes

Sentence-transformer models for embedding generation run exclusively on laptop-01 and laptop-02. They never run on fleet workers, the controller, or single-board computers.

**Why:** Embedding models (e.g., `all-MiniLM-L6-v2`, `bge-base-en-v1.5`) require substantial memory and CPU. Running them on a 1GB Raspberry Pi would either fail or degrade the node's ability to handle its primary workload. Dedicating embedding to capable hardware ensures predictable throughput and prevents resource contention.

**Implementation:** Embedding workers query the database for pending documents, generate embeddings locally, and write results back through the REST API. The queue system (Redis-backed, with Lua-scripted atomic operations) prevents duplicate processing.

### 3.4 Zero Overlap Between Review and Meta-Review Panels

When a review pipeline evaluates an artifact (code, documentation, research findings), the models that produce findings and the models that adversarially challenge those findings must be completely disjoint sets.

```
  Review Panel              Meta-Review Panel
  ─────────────             ──────────────────
  Opus                      Fable
  Sonnet                    Hermes-405B
  DeepSeek-Chat             Nemotron-Ultra-550B
  Qwen3-Coder               Qwen3-Next-80B

  Intersection: EMPTY (enforced)
```

**Why:** If Opus produces a finding and Opus also meta-reviews it, the meta-review is not independent. The model's systematic biases (toward certain code patterns, away from certain false-positive categories) are present in both the finding and the evaluation of that finding. Zero overlap guarantees that meta-review introduces genuinely new perspectives.

**Implementation:** Panel definitions are constants in the workflow file. A runtime assertion verifies that the intersection of review and meta-review model sets is empty before execution begins.

### 3.5 Triple-Store All Knowledge

Every piece of knowledge the system acquires is stored in three complementary data stores:

| Store | Technology | Purpose | Query Pattern |
|-------|-----------|---------|---------------|
| Relational + Vector | PostgreSQL + pgvector | Structured data, similarity search | SQL + cosine similarity |
| Cache + Queue | Redis (Sentinel HA) | Hot data, rate limiting, job queues | Key-value, sorted sets |
| Graph | OrientDB | Relationships, topology, lineage | Graph traversal, path queries |

**Why:** Each store excels at a different query pattern. Finding "documents similar to X" is a vector operation (PostgreSQL + pgvector). Finding "which models reviewed which artifacts" is a graph traversal (OrientDB). Caching consensus results to avoid redundant multi-model calls is a TTL key-value operation (Redis). Using a single store for all three patterns would mean poor performance on at least two of them.

**Implementation:** The REST API gateway mediates all writes, ensuring that a single logical write fans out to all three stores. Reads are routed to the store best suited for the query pattern. An intelligent search endpoint (`/search/intelligent`) performs hybrid search across all stores and merges results using Reciprocal Rank Fusion (RRF, k=60).

### 3.6 Everything Is Evolved, Not Hardcoded

System parameters -- consensus thresholds, team compositions, routing weights, retry budgets -- are not set by engineers in configuration files. They are evolved by genetic algorithms that optimize for downstream outcomes.

**Why:** The parameter space is too large and too interdependent for manual tuning. The optimal consensus threshold depends on the task type, which depends on the model panel, which depends on the available models, which changes as providers update their offerings. A genetic algorithm explores this space continuously and adapts to changes without human intervention.

**Implementation:** The GA engine (`learning/ga_engine.py`) maintains a population of configuration genomes. Each genome encodes a complete parameter set. Fitness is evaluated by running the configuration against a held-out set of tasks and measuring outcome quality. The population evolves through tournament selection, crossover, and mutation. The current best genome is applied to production traffic. Specialized GA optimizers exist for model routing (`tools/genetic_model_optimizer.py`), RAG retrieval (`tools/ga_rag_retrieval_optimizer.py`), team selection (`tools/ga_team_selection_fixed.py`), and training data curation (`tools/ga_training_data_curator.py`).

**Tradeoff acknowledged:** GA evolution is non-deterministic. Two runs from the same initial population may converge to different optima. The system mitigates this by maintaining the top-k genomes (not just the single best) and periodically reseeding the population with known-good configurations to prevent drift.

---

## 4. Overall System Architecture

### Level 1: Executive Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENTS                                    │
│  Claude Code  |  Workflows (.mjs)  |  REST Clients  |  Cron Jobs   │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             │ HTTP (port 5000)
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│                    ORCHESTRATOR (aio-01)                             │
│                                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ Classify  │ │  Route   │ │Distribute│ │ Collect  │ │Synthesize│ │
│  │          ├─►          ├─►          ├─►          ├─►          │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
└──────┬───────────────────────────┬──────────────────────────┬──────┘
       │                           │                          │
       │ SSH                       │ SSH                      │ SSH
       │                           │                          │
┌──────▼──────┐  ┌────────────────▼──────────┐  ┌────────────▼──────┐
│   WORKERS   │  │        WORKERS            │  │     WORKERS       │
│  server-01  │  │  server-02  |  server-03  │  │  pi-01  |  pi-02  │
│  laptop-01  │  │  desktop-ap |  server-ap  │  │                   │
└──────┬──────┘  └────────────────┬──────────┘  └────────────┬──────┘
       │                          │                           │
       │ HTTPS                    │ HTTPS                     │ HTTPS
       │                          │                           │
┌──────▼──────────────────────────▼───────────────────────────▼──────┐
│                      MODEL PROVIDERS                                │
│  OpenRouter (150+)  |  Anthropic  |  Google  |  Groq  |  DeepSeek  │
│  Cerebras  |  Cohere  |  xAI  |  Others                            │
└──────┬──────────────────────────────────────────────────────┬──────┘
       │                                                      │
       │ (Results flow back up through the same path)         │
       │                                                      │
┌──────▼──────────────────────────────────────────────────────▼──────┐
│                       DATA LAYER (aio-01)                          │
│  PostgreSQL+pgvector (5433)  |  Redis Sentinel (6379)  |  OrientDB │
└────────────────────────────────────────────────────────────────────┘
```

### Level 2: Major Components with Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           ORCHESTRATOR (aio-01)                          │
│                                                                          │
│  ┌────────────────┐     ┌─────────────────┐     ┌───────────────────┐  │
│  │ Task Classifier │     │  Model Router    │     │ Fleet Dispatcher  │  │
│  │                 │     │                  │     │                   │  │
│  │ 15 categories   ├────►│ Thompson Sampling├────►│ SSH orchestrator  │  │
│  │ keyword+phrase  │     │ capability matrix│     │ circuit breakers  │  │
│  │ context scoring │     │ tier weights     │     │ health monitoring │  │
│  └────────────────┘     └─────────────────┘     └────────┬──────────┘  │
│                                                           │             │
│  ┌────────────────┐     ┌─────────────────┐              │             │
│  │ Consensus Eng. │     │ Weighted Voting  │              │             │
│  │                │     │                  │              │             │
│  │ 5 strategies   │◄────┤ tier x cap x     │              │             │
│  │ arbiter synth. │     │ conf x hist x    │              │             │
│  │ disagreement   │     │ calibration      │              │             │
│  └───────┬────────┘     └─────────────────┘              │             │
│          │                                                │             │
│  ┌───────▼────────┐     ┌─────────────────┐     ┌───────▼───────────┐  │
│  │ Result Synth.  │     │ Rate Limiter     │     │ Worker Registry   │  │
│  │                │     │                  │     │                   │  │
│  │ arbiter-based  │     │ per-provider     │     │ heartbeat monitor │  │
│  │ majority vote  │     │ token budgets    │     │ capability track  │  │
│  │ weighted merge │     │ cost estimation  │     │ failure detection │  │
│  └────────────────┘     └─────────────────┘     └───────────────────┘  │
│                                                                          │
│  ┌────────────────┐     ┌─────────────────┐     ┌───────────────────┐  │
│  │ Queue System   │     │ Knowledge Store  │     │ Intelligent Search│  │
│  │                │     │                  │     │                   │  │
│  │ 4-stage pipe:  │     │ hybrid search    │     │ RRF fusion across │  │
│  │ store->chunk   │     │ pgvector cosine  │     │ 4 sources: KB,    │  │
│  │ ->embed->graph │     │ tsvector/tsquery │     │ memory, exp, graph│  │
│  │ Redis+Lua      │     │ cascading embed  │     │                   │  │
│  └────────────────┘     └─────────────────┘     └───────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
         │              │                │              │
         │ SQL          │ Redis          │ REST         │ Gremlin
         ▼              ▼                ▼              ▼
┌──────────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────────┐
│  PostgreSQL  │ │   Redis    │ │   OrientDB   │ │  Prometheus  │
│  + pgvector  │ │  Sentinel  │ │              │ │  + Grafana   │
│              │ │            │ │  Graph DB    │ │              │
│ learning.*   │ │ Queues     │ │  Infra graph │ │  Metrics     │
│ workflow.*   │ │ Cache      │ │  Knowledge   │ │  Dashboards  │
│ monitoring.* │ │ Rate limit │ │  Lineage     │ │  Alerts      │
│ costs.*      │ │ Locks      │ │              │ │              │
└──────────────┘ └────────────┘ └──────────────┘ └──────────────┘
```

### Level 3: Subsystem Interactions (API Call Flow)

This diagram traces a single code review request through the system, showing every API call between components.

```
Client                 Orchestrator           Fleet              Providers         Data Layer
  │                       │                     │                    │                 │
  │  POST /review         │                     │                    │                 │
  │──────────────────────►│                     │                    │                 │
  │                       │                     │                    │                 │
  │                       │ classify(prompt)     │                    │                 │
  │                       │──┐                  │                    │                 │
  │                       │  │ task_type:        │                    │                 │
  │                       │  │ code_review       │                    │                 │
  │                       │◄─┘                  │                    │                 │
  │                       │                     │                    │                 │
  │                       │ GET /workers        │                    │                 │
  │                       │────────────────────►│                    │                 │
  │                       │ [healthy workers]   │                    │                 │
  │                       │◄────────────────────│                    │                 │
  │                       │                     │                    │                 │
  │                       │ check circuit       │                    │                 │
  │                       │ breakers            │                    │                 │
  │                       │──┐                  │                    │                 │
  │                       │◄─┘ [open circuits   │                    │                 │
  │                       │     excluded]        │                    │                 │
  │                       │                     │                    │                 │
  │                       │ SSH: execute(task)   │                    │                 │
  │                       │────────────────────►│ (4 workers,        │                 │
  │                       │                     │  parallel)          │                 │
  │                       │                     │                    │                 │
  │                       │                     │ POST /completions  │                 │
  │                       │                     │───────────────────►│                 │
  │                       │                     │ [model response]   │                 │
  │                       │                     │◄───────────────────│                 │
  │                       │                     │                    │                 │
  │                       │ [4 worker results]  │                    │                 │
  │                       │◄────────────────────│                    │                 │
  │                       │                     │                    │                 │
  │                       │ weighted_vote()     │                    │                 │
  │                       │──┐                  │                    │                 │
  │                       │  │ calculate weights │                    │                 │
  │                       │  │ tier x cap x conf│                    │                 │
  │                       │  │ x hist x calib   │                    │                 │
  │                       │◄─┘                  │                    │                 │
  │                       │                     │                    │                 │
  │                       │ arbiter_synthesize() │                    │                 │
  │                       │────────────────────────────────────────►│                 │
  │                       │ [synthesized result] │                    │                 │
  │                       │◄────────────────────────────────────────│                 │
  │                       │                     │                    │                 │
  │                       │ store results       │                    │                 │
  │                       │───────────────────────────────────────────────────────────►│
  │                       │                     │                    │                 │
  │  {findings, meta}     │                     │                    │                 │
  │◄──────────────────────│                     │                    │                 │
  │                       │                     │                    │                 │
```

### Component Reference Table

| Component | Technology | Location | Role |
|-----------|-----------|----------|------|
| Orchestrator API | FastAPI / Flask | `api/`, `admin-api/` | Central request gateway, task routing |
| Task Classifier | JavaScript | `shared/task-type-detector.js` | Categorizes prompts into 15 task types |
| Model Router | Python | `orchestrate_smart.py` | Thompson Sampling + capability matrix routing |
| Fleet Dispatcher | Python/JS | `shared/fleet-ssh-orchestrator.js`, `shared/fleet_executor.py` | SSH-based task distribution to workers |
| Dynamic Worker Pool | JavaScript | `fleet/dynamic-worker-pool.js` | Health-checked worker discovery with circuit breakers |
| Worker Registry | FastAPI | `admin-api/worker-registry-service.py` | PostgreSQL-backed worker state tracking |
| Consensus Engine | JavaScript | `shared/consensus-engine.js` | 5-strategy consensus with arbiter synthesis |
| Weighted Voting | JavaScript | `shared/weighted-voting.cjs` | Multi-factor vote weighting with BFT |
| Circuit Breaker | JavaScript | `shared/circuit-breaker.cjs` | Per-model/per-worker failure isolation |
| Queue System | Flask Blueprint | `api/queue_redis_blueprint.py` | 4-stage Redis pipeline with Lua atomicity |
| Knowledge Store | Flask Blueprint | `api/knowledge.py` | Hybrid pgvector + tsvector search |
| Intelligent Search | Flask Blueprint | `api/intelligent_search.py` | RRF fusion across 4 data sources |
| GA Engine | Python | `learning/ga_engine.py` | Genetic algorithm for parameter evolution |
| Model Optimizer | Python | `tools/genetic_model_optimizer.py` | GA-based model routing evolution |
| Team Selector | Python | `tools/ga_team_selection_fixed.py` | GA-based review panel composition |
| RAG Optimizer | Python | `tools/ga_rag_retrieval_optimizer.py` | GA-based retrieval parameter tuning |
| Feedback Loop Monitor | Python/JS | `tools/feedback_loop_optimizer.py`, `shared/feedback-loop-adapter.cjs` | 4-layer self-referential bias detection |
| Prompt Enhancer | Python | `shared/prompt_enhancer.py` | Task-aware prompt augmentation |
| Fleet Topology | JavaScript | `shared/fleet-topology.js`, `shared/fleet-topology-dynamic.js` | Static and dynamic fleet graph |
| Workflow Storage | JavaScript | `shared/workflow-storage-adapter.js` | PostgreSQL-backed workflow tracking |
| PostgreSQL Adapter | JavaScript/Python | `shared/postgres-adapter.js`, `shared/postgres_adapter.py` | Database access abstraction |
| Evaluation Harness | JavaScript | `~/.claude/self/evaluation-harness.mjs` | Adversarial 5-dimension evaluation |

---

## 5. Distributed Fleet

### 5.1 Fleet Topology

The fleet consists of 9 nodes arranged in a controller-worker topology. One node (aio-01) serves as the controller and orchestrator. Eight nodes serve as workers. The controller never executes worker tasks; workers never run database operations or embedding generation.

```
                    ┌─────────────────────────────────┐
                    │        CONTROLLER (aio-01)       │
                    │   2 cores, 7GB RAM, x86_64       │
                    │                                   │
                    │   Orchestrator API (:5000)        │
                    │   Worker Registry  (:8002)        │
                    │   PostgreSQL       (:5433)        │
                    │   Redis Sentinel   (:6379)        │
                    │   OrientDB         (:2424)        │
                    │                                   │
                    │   NEVER runs worker tasks         │
                    └───────────────┬───────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
           ┌────────▼──────┐ ┌─────▼────────┐ ┌───▼──────────┐
           │  HIGH TIER    │ │  MID TIER    │ │  LOW TIER    │
           │               │ │              │ │              │
           │ server-01     │ │ laptop-01    │ │ pi-01        │
           │  8C, 16GB     │ │  4C, 31GB    │ │  ARM64, 1GB  │
           │  x86_64       │ │  x86_64      │ │  aarch64     │
           │               │ │  (+ embed)   │ │              │
           │ server-02     │ │              │ │ pi-02        │
           │  8C, 32GB     │ │ desktop-ap   │ │  ARM64, 1GB  │
           │  x86_64       │ │  1GB         │ │  aarch64     │
           │               │ │  armv7l      │ │              │
           │ server-03     │ │              │ └──────────────┘
           │  8C, 32GB     │ │ server-ap    │
           │  x86_64       │ │  1GB         │
           │               │ │  armv7l      │
           └───────────────┘ └──────────────┘
```

**Hardware heterogeneity** is a deliberate design choice. The fleet spans three CPU architectures: x86_64 (servers), ARM64/aarch64 (Raspberry Pis), and ARMv7/armhf (router-based workers running DD-WRT with Debian chroot). Every worker runs the same fleet executor script (`shared/fleet_executor.py`), which uses only Python standard library modules and routes all model requests through the proxy on aio-01. The executor's architecture-agnostic design means it functions identically on x86_64, ARM64, and ARMv7 without dependency installation. Fleet deployment is automated via Ansible with 11 roles (common, nfs, postgresql, redis, unified_api, monitoring, worker, ga_tools, scraping, orientdb, backup) and a 10-phase site playbook.

### 5.2 Why SSH Instead of Kubernetes

The system uses SSH for task distribution rather than Kubernetes, a message queue (RabbitMQ, Kafka), or a container orchestration platform. This was a considered decision.

**Arguments for Kubernetes:**
- Standardized deployment model (containers)
- Built-in health checking and restart policies
- Service discovery and load balancing
- Ecosystem of operators and tools

**Arguments against Kubernetes (for this system):**
- **Overhead:** Kubernetes requires a control plane (etcd, API server, scheduler, controller-manager) that would consume most of aio-01's 7GB RAM before any worker tasks execute.
- **Heterogeneous hardware:** Running Kubernetes on 1GB ARM boards is possible but fragile. The kubelet alone consumes 200-500MB.
- **Task model mismatch:** This system's tasks are short-lived (5-60 seconds), stateless API calls. Kubernetes is optimized for long-running services. The pod startup overhead (image pull, container creation, network setup) would exceed the task duration for simple consensus queries.
- **Complexity budget:** SSH is a dependency every node already has. Kubernetes introduces a significant operational surface (certificate rotation, etcd backup, version upgrades, CNI plugin management) that adds no value for this workload.

**Arguments for message queues:**
- Decoupled producers and consumers
- Guaranteed delivery
- Backpressure handling

**Arguments against message queues (for this system):**
- **Additional infrastructure:** RabbitMQ or Kafka would require dedicated nodes or compete for resources on aio-01.
- **Latency:** Enqueue-dequeue-acknowledge adds 10-50ms per hop. For a 4-model consensus query, this compounds to 40-200ms of queue overhead.
- **Complexity:** The system's task distribution pattern is simple: the controller knows the worker set, health-checks them, and dispatches tasks. A message queue adds consumer group management, dead-letter handling, and partition rebalancing for a problem that SSH solves directly.

**The SSH approach:**

```
Controller                          Worker
    │                                  │
    │  ssh claude@worker-host \        │
    │    'python3 fleet_executor.py    │
    │     --model opus                 │
    │     --task <base64-encoded>'     │
    │────────────────────────────────►│
    │                                  │ Decode task
    │                                  │ Call model API
    │                                  │ Return JSON result
    │  {result: ..., tokens: ...,      │
    │   duration_ms: ...}              │
    │◄────────────────────────────────│
    │                                  │
```

SSH provides: authentication (key-based), encryption (in transit), multiplexing (ControlMaster), and timeout handling (ConnectTimeout=5s). The fleet executor validates hostnames against an allowlist to prevent command injection. Task prompts are base64-encoded for shell safety.

**Tradeoff acknowledged:** SSH does not provide guaranteed delivery. If a worker crashes mid-task, the result is lost. The circuit breaker (Section 5.4) detects this and retries on another worker. For the system's workload -- stateless API calls that can be safely retried -- this is acceptable.

### 5.3 Worker Lifecycle

Workers follow a registration-heartbeat-execution lifecycle managed by the Worker Registry service.

```
                   ┌──────────┐
                   │UNREGISTER│
                   └──────────┘
                        │
              POST /register
              (hostname, ip, cores,
               ram, arch, capabilities)
                        │
                        ▼
                   ┌──────────┐
            ┌─────│  ACTIVE   │◄─────────────────────────────────┐
            │     └──────────┘                                    │
            │          │                                          │
            │   POST /heartbeat/{host}                            │
            │   (every 60s)                                       │
            │          │                                          │
            │          ▼                                          │
            │     ┌──────────┐                                    │
            │     │  ACTIVE   │────────────────┐                  │
            │     └──────────┘                 │                  │
            │          │                       │                  │
            │   failure count                  │                  │
            │   reaches 1-2                    │                  │
            │          │                  heartbeat               │
            │          ▼                  received                │
            │     ┌──────────┐            after recovery          │
            │     │ DEGRADED  │────────────────┘                  │
            │     └──────────┘                                    │
            │          │                                          │
            │   failure count                                     │
            │   reaches 3+                                        │
            │          │                                  manual   │
            │          ▼                                  reset    │
            │     ┌──────────┐                                    │
            └────►│  FAILED   │───────────────────────────────────┘
                  └──────────┘
```

**State transitions:**

| From | To | Trigger | Effect |
|------|----|---------|--------|
| Unregistered | Active | `POST /register` with valid capabilities | Worker added to registry, available for tasks |
| Active | Active | `POST /heartbeat/{host}` within 2 minutes | Timestamp updated, failure count reset |
| Active | Degraded | 1-2 consecutive task failures | Worker deprioritized in routing, not excluded |
| Degraded | Active | Successful heartbeat after recovery period | Failure count reset, full priority restored |
| Degraded | Failed | 3+ consecutive task failures | Worker excluded from task routing |
| Failed | Active | Manual reset via admin API | Failure count reset, re-enters active pool |
| Any | Failed | No heartbeat for >2 minutes | Worker presumed down, excluded from routing |

The Worker Registry stores state in the `fleet.workers` PostgreSQL table. State transitions use `SELECT FOR UPDATE` to prevent race conditions when multiple concurrent tasks report failures for the same worker.

### 5.4 Circuit Breaker

The circuit breaker provides per-model and per-worker failure isolation. It prevents cascading failures by stopping requests to failing endpoints before they accumulate timeouts.

```
        ┌─────────────────────────────────────────────────┐
        │                                                  │
        │   ┌────────┐    5 failures    ┌────────┐        │
        │   │ CLOSED  │───────────────►│  OPEN   │        │
        │   │         │                │         │        │
        │   │ Normal  │                │ All     │        │
        │   │ traffic │   ┌────────┐   │ calls   │        │
        │   │ flows   │◄──│  HALF  │◄──│ blocked │        │
        │   │         │   │  OPEN  │   │         │        │
        │   └────────┘   │        │   └────────┘        │
        │     ▲   2      │ Allow  │      │   30s         │
        │     │ success  │ 3 test │      │  timeout      │
        │     │ in test  │ calls  │      │               │
        │     └──────────│        │◄─────┘               │
        │                │ any    │                       │
        │                │ failure│                       │
        │                │  ──►   │──────►OPEN            │
        │                └────────┘                       │
        │                                                  │
        └─────────────────────────────────────────────────┘
```

**Configuration:**

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `failure_threshold` | 5 | Consecutive failures before circuit opens |
| `timeout_ms` | 30,000 | Time in OPEN state before transitioning to HALF-OPEN |
| `half_open_max_calls` | 3 | Maximum test calls allowed in HALF-OPEN |
| `half_open_success_threshold` | 2 | Successes required in HALF-OPEN to close circuit |

**Implementation details:**

- State is tracked per-model in an in-memory object (not persisted across restarts -- by design, since a restart clears transient failures).
- State transitions are logged to `monitoring.circuit_breaker_events` in PostgreSQL for post-incident analysis.
- When a circuit opens, a webhook notification is sent via `webhook-notifier.cjs` to alert operators.
- The `filterAvailableModels()` function integrates circuit breaker state with the model routing layer, automatically excluding models with open circuits from candidate pools.
- A rolling history of the last 10 calls per model is maintained for diagnostic inspection.

### 5.5 Health Monitoring

The controller performs continuous health monitoring of the fleet through two complementary mechanisms.

**Active health checks** (Dynamic Worker Pool):
- Every 10 seconds, the controller sends HTTP GET requests to port 7340 on each worker.
- Workers must respond with `{status: 'ok', node: hostname, session: {...}}` within 2 seconds.
- Responses larger than 10KB are rejected to prevent OOM conditions in the health checker.
- Failed health checks increment the circuit breaker failure counter.
- The worker pool maintains a 10-second TTL cache of health results, with a stale-cache fallback window of 5 minutes.

**Passive heartbeat monitoring** (Worker Registry):
- Workers send `POST /heartbeat/{hostname}` to the registry every 60 seconds.
- The registry marks workers as failed if no heartbeat arrives within 2 minutes.
- Heartbeats include current load, available memory, and active task count.

**Fleet status** is exposed via the Prometheus exporter and visualized in Grafana dashboards. Node-level metrics (CPU, memory, disk, network) are collected by `node_exporter` on port 9100 of each worker and scraped by Prometheus every 15 seconds.

### 5.6 Worker Capabilities and Constraints

Workers declare their capabilities at registration time. The dispatcher uses these declarations to match tasks to appropriate workers.

**Capability categories:**

| Capability | Description | Typical Workers |
|-----------|-------------|-----------------|
| `code-review` | Code analysis, bug detection | server-01, server-02, server-03, laptop-01 |
| `security-scan` | Security vulnerability analysis | server-01, server-02, server-03 |
| `research` | Web search, document analysis | All workers |
| `web-learning` | Web scraping, knowledge extraction | server-01, server-02, server-03 |
| `code-analysis` | Static analysis, architecture review | server-01, server-02, server-03, laptop-01 |
| `pdf-processing` | PDF extraction and analysis | server-02, server-03 (high memory) |
| `doc-generation` | Documentation writing | All workers |
| `test` | Test generation and execution | server-01, server-02, server-03 |

**Hard constraints (enforced by the architecture):**

1. **aio-01 never runs worker tasks.** It is the controller, database host, and API gateway. Running worker tasks on it would create resource contention with these critical services.
2. **Workers never run embedding models.** Sentence-transformer models run exclusively on laptop-01 and laptop-02, which have sufficient RAM (31GB each). Embedding on a 1GB Raspberry Pi would either crash or take prohibitively long.
3. **Workers never access databases directly.** All data access flows through the REST API on aio-01, ensuring centralized connection pooling, access control, and audit logging.

---

## 6. Orchestration Layer

### 6.1 API Architecture

The orchestration layer is implemented as a collection of independent services, each running on its own port on the controller node (aio-01). This is a microservice-style decomposition, not a monolithic Flask application.

**Service inventory:**

| Service | Framework | Port | Purpose |
|---------|-----------|------|---------|
| Orchestrator API | FastAPI | 5000 | Primary gateway, task routing, model selection |
| Admin API | FastAPI | 8001 | Maintenance operations, authentication management |
| Worker Registry | FastAPI | 8002 | Worker state tracking, heartbeat processing |
| Fleet Control | FastAPI | 8004 | Remote task execution, SSH dispatch |
| Queue (Redis) | Flask Blueprint | (registered) | 4-stage document processing pipeline |
| Knowledge | Flask Blueprint | (registered) | Hybrid search, embedding management |
| Intelligent Search | Flask Blueprint | (registered) | Cross-store RRF fusion search |

The Flask blueprints are modular components designed to be registered into a host application. Each blueprint encapsulates its routes, error handlers, and database access patterns. The blueprint architecture allows composition: a deployment can register only the blueprints it needs.

**Blueprint registration pattern:**

```python
from flask import Flask
from api.queue_redis_blueprint import queue_bp
from api.knowledge import knowledge_bp
from api.intelligent_search import intelligent_search_bp

app = Flask(__name__)
app.register_blueprint(queue_bp)        # /queue/*
app.register_blueprint(knowledge_bp)    # /knowledge/*
app.register_blueprint(intelligent_search_bp)  # /search/*
```

### 6.2 Task Classification

When a request arrives at the orchestrator, the first step is classifying the task to determine routing, model selection, and consensus strategy. The Task Type Detector (`shared/task-type-detector.js`) categorizes prompts into 15 task types.

**Task categories:**

| Category | Keywords/Phrases | Recommended Strategy | Min Models |
|----------|-----------------|---------------------|------------|
| `code_generation` | "write", "implement", "create function" | `weighted` | 3 |
| `code_review` | "review", "check code", "find bugs" | `weighted` | 4 |
| `bug_detection` | "bug", "error", "crash", "failing" | `unanimous_for_critical` | 4 |
| `security_audit` | "security", "vulnerability", "CVE" | `conservative_consensus` | 3 |
| `architecture_review` | "architecture", "design", "scalability" | `weighted` | 3 |
| `research` | "research", "investigate", "explore" | `majority` | 3 |
| `fact_checking` | "verify", "fact check", "is it true" | `pairwise` | 4 |
| `consensus` | "agree", "consensus", "vote" | `majority` | 5 |
| `creative_writing` | "write story", "creative", "narrative" | `single` | 1 |
| `math_reasoning` | "calculate", "prove", "derive" | `unanimous_for_critical` | 3 |
| `documentation` | "document", "README", "API docs" | `single` | 2 |
| `testing` | "test", "unit test", "integration test" | `weighted` | 3 |
| `optimization` | "optimize", "performance", "speed" | `weighted` | 3 |
| `refactoring` | "refactor", "clean up", "simplify" | `weighted` | 3 |
| `data_analysis` | "analyze data", "statistics", "trends" | `majority` | 3 |

The classifier uses keyword matching, phrase detection, context scoring, and negative keyword exclusion. Each category has a weight multiplier that adjusts confidence scores. The `detectTaskType()` function returns the primary type, confidence level, secondary types, and per-category scores.

**Classification also determines arbiter configuration.** For example, `security_audit` tasks use conservative consensus with 3+ models because security findings have asymmetric cost -- a missed vulnerability is far more expensive than a false positive. Conversely, `creative_writing` uses a single model because creative tasks have no objective "correct" answer that benefits from consensus.

### 6.3 Request Lifecycle

A request follows a six-stage pipeline through the orchestration layer.

```
  ┌─────────┐   ┌─────────┐   ┌───────────┐   ┌─────────┐   ┌───────────┐   ┌────────┐
  │CLASSIFY  │──►│  ROUTE  │──►│DISTRIBUTE │──►│ COLLECT │──►│SYNTHESIZE │──►│ RETURN │
  │          │   │         │   │           │   │         │   │           │   │        │
  │Detect    │   │Select   │   │SSH to     │   │Await    │   │Arbiter or │   │Format  │
  │task type │   │models   │   │workers    │   │results  │   │vote       │   │response│
  │Set       │   │Check    │   │Circuit    │   │Handle   │   │Weighted   │   │Store   │
  │strategy  │   │circuits │   │breakers   │   │timeouts │   │merge      │   │metrics │
  └─────────┘   └─────────┘   └───────────┘   └─────────┘   └───────────┘   └────────┘
```

**Stage 1: Classify.** The task classifier analyzes the prompt text, determines the task category, and selects the appropriate consensus strategy, minimum model count, and arbiter configuration. This stage adds 1-5ms of latency.

**Stage 2: Route.** The model router selects which models to query based on three inputs: (a) the task type's capability requirements, (b) the Thompson Sampling bandit state (historical success rates per model per task type), and (c) circuit breaker state (excluding models with open circuits). The router uses a three-strategy layered selection:

1. **Thompson Sampling** (contextual bandit): Draws from Beta distributions parameterized by each model's success/failure history for the detected task type. Balances exploration (trying underused models) with exploitation (favoring proven models).
2. **Capability matrix query**: Queries `learning.model_capabilities` in PostgreSQL for quality scores per model per task category.
3. **GA-evolved auto-profiler**: For novel task types with insufficient bandit data, the genetic algorithm suggests model combinations based on evolved fitness scores.

For complex tasks, Thompson Sampling is forced with strong models. For simple, high-confidence tasks, the router may select cheaper models to reduce cost.

**Stage 3: Distribute.** The fleet dispatcher sends tasks to selected workers via SSH. All workers execute concurrently. The dispatcher enforces a maximum of 6 concurrent workers to prevent overwhelming the network and provider rate limits. Task prompts are base64-encoded for shell safety. Hostnames are validated against an allowlist before SSH connections are established.

**Stage 4: Collect.** The orchestrator awaits results from all dispatched workers with per-worker timeouts. If a worker exceeds its timeout or returns an error, the circuit breaker records the failure. Successful results are collected into an array for synthesis. If fewer than the minimum required results arrive (based on the task type's `minModels` configuration), the orchestrator retries with alternative workers.

**Stage 5: Synthesize.** Results are synthesized using the strategy selected in Stage 1. For arbiter-based strategies, the arbiter model receives all worker responses and produces a synthesized output. For vote-based strategies, the weighted voting module calculates per-response weights and determines the majority position. See Sections 7 (Consensus Engine) and 8 (Multi-AI Review Pipeline) for detailed synthesis mechanics.

**Stage 6: Return.** The final result is formatted, metadata is attached (models used, latencies, token counts, confidence scores), and the response is returned to the client. Execution metrics are stored in `monitoring.execution_summary` for cost tracking and performance analysis. Workflow metadata is stored via the workflow storage adapter.

### 6.4 Circuit Breaker Integration

The orchestration layer integrates circuit breakers at two levels:

1. **Per-worker circuit breakers** (in `fleet/dynamic-worker-pool.js`): Track SSH connectivity failures to worker nodes. Threshold: 3 failures, 60-second reset. When a worker's circuit opens, it is excluded from the available worker pool entirely.

2. **Per-model circuit breakers** (in `shared/circuit-breaker.cjs`): Track API call failures to specific models. Threshold: 5 failures, 30-second reset. When a model's circuit opens, the routing layer excludes it from candidate model sets.

These two levels operate independently. A worker can be healthy (closed worker circuit) while a model routed through it can be failing (open model circuit), or vice versa. The orchestrator intersects both circuit states to determine the viable execution matrix.

### 6.5 Rate Limiting

Rate limiting operates at two granularities:

**Per-provider rate limiting** is enforced at the orchestration layer to respect provider-imposed quotas. Each provider (OpenRouter, Anthropic, Google, Groq, etc.) has configured limits for requests per minute and tokens per minute. The rate limiter uses a sliding-window counter backed by Redis. When a provider's limit is approached, the router either queues the request (if the task can tolerate delay) or routes to an alternative provider (if the task is latency-sensitive).

**Per-API-key rate limiting** is enforced at the REST API layer for external clients. Rate limits are stored in the `auth.api_keys` table and enforced using atomic read-modify-write operations on `monitoring.rate_limit_state` in PostgreSQL. The implementation uses minute-window bucketing (`INSERT...ON CONFLICT` with window truncation) for atomic, race-condition-free limit enforcement. Exceeded limits return HTTP 429 with a `Retry-After` header.

Health and metrics endpoints (`/health`, `/metrics`) are exempt from rate limiting to ensure monitoring systems always receive responses.

### 6.6 Weighted Voting

When the consensus strategy is `weighted`, vote weights are calculated using a five-factor formula:

```
weight = tier_weight x capability_score x confidence x historical_accuracy x calibration_penalty
```

**Factor definitions:**

| Factor | Source | Range | Purpose |
|--------|--------|-------|---------|
| `tier_weight` | Static mapping | 0.50 - 1.00 | Base model capability tier (Opus=1.0, Haiku=0.6) |
| `capability_score` | Per-task matrix | 0.0 - 1.0 | Task-specific model strength |
| `confidence` | Model self-report | 0.0 - 1.0 | Model's stated confidence in its response |
| `historical_accuracy` | Thompson Sampling | 0.0 - 1.0 | `avg_quality` from bandit state |
| `calibration_penalty` | Confidence audit | 0.25 - 1.0 | Penalty for confidence/accuracy mismatch |

**Model tier weights (representative values):**

| Model | Tier Weight | Rationale |
|-------|-------------|-----------|
| Opus | 1.00 | Highest general capability |
| Fable | 0.95 | Near-Opus quality on most tasks |
| GPT-4o | 0.95 | Comparable to Opus on structured tasks |
| Gemini | 0.90 | Strong reasoning, slightly below Opus |
| Sonnet | 0.85 | High capability, balanced speed/quality |
| DeepSeek-Coder | 0.70 | Strong on code, weaker on general tasks |
| Mistral-7B | 0.65 | Smaller model, competitive on simple tasks |
| Haiku | 0.60 | Fast, lower capability, useful for verification |
| Phi-4-mini | 0.50 | Smallest model, cost-efficient for routing |

The capability matrix provides per-model, per-task-category scores. For example, `deepseek-coder` scores 1.0 on `code_generation` but 0.5 on `research`. This matrix is seeded with initial estimates and refined by the GA model optimizer based on observed outcomes.

The calibration penalty addresses "lying models" -- models that consistently report high confidence on incorrect answers. The penalty is computed as the ratio of a model's average accuracy to its average stated confidence. A model that claims 95% confidence but is accurate 60% of the time receives a calibration penalty of approximately 0.63.

**Byzantine Fault Tolerance:** The weighted voting module includes defenses against adversarial or corrupted model responses:
- **Weighted median:** Used instead of weighted mean for numeric outputs (robust to outliers).
- **Trimmed mean:** Drops the top and bottom 20% of responses before averaging.
- **MAD outlier detection:** Flags responses whose distance from the median exceeds 3x the Median Absolute Deviation.
- **Sybil detection:** Detects vote flooding from a single model family (e.g., if three Llama variants produce identical responses, they are treated as a single vote).

---

## 7. Consensus Engine

### 7.1 Overview

The consensus engine is the mechanism by which multiple model responses are combined into a single output. It supports five strategies, each appropriate for different task types and confidence requirements. The engine is implemented in `shared/consensus-engine.js` with a default worker panel of six models: Sonnet, Opus, Haiku, GPT-4o, Gemini, and Cerebras-120B. An optional seventh worker (OpenClaw, execution-backed verification) can be enabled via the `OPENCLAW_ENABLED` feature flag.

The consensus engine handles three decision types:

| Decision Type | Question Being Answered | Example |
|--------------|------------------------|---------|
| `issue` | "Is this a real bug?" | Code review finding validation |
| `pr` | "Should this PR be approved?" | Pull request evaluation |
| `fix` | "Which fix approach is best?" | Fix proposal selection |

### 7.2 Consensus Strategies

#### Rotating Arbiter

```
  Worker 1 ──┐
  Worker 2 ──┤
  Worker 3 ──┼──► Arbiter (rotates: sonnet → opus → haiku → gpt-4o → ...)
  Worker 4 ──┤        │
  Worker 5 ──┤        ▼
  Worker 6 ──┘   Synthesized result
```

The arbiter rotates between available models on each invocation using a modular index: `arbiterRotationIndex % availableModels.length`. The rotation pool includes: Sonnet, Opus, Haiku, GPT-4o, Gemini, and Cerebras-120B.

**When to use:** Default strategy for most tasks. Rotation prevents any single model from dominating synthesis, reducing systematic arbiter bias over time.

**Tradeoff:** An individual synthesis may be lower quality than always using the strongest arbiter (Opus). The system accepts this per-query variance to achieve lower long-term arbiter bias.

#### Single Arbiter

A fixed arbiter (default: Sonnet) synthesizes all worker responses. No rotation occurs.

**When to use:** Tasks where synthesis consistency matters more than bias mitigation. Documentation generation, where stylistic consistency across outputs is valued, uses single-arbiter.

**Tradeoff:** Introduces systematic arbiter bias. Sonnet's preferences (e.g., for structured output, against verbose explanations) color every synthesis. Acceptable for tasks where this bias is benign.

#### Majority Vote

```
  Worker 1: is_real_issue = true  ──┐
  Worker 2: is_real_issue = true  ──┤
  Worker 3: is_real_issue = false ──┼──► Count: 4 true, 2 false
  Worker 4: is_real_issue = true  ──┤    Consensus: 67% (>= 60%)
  Worker 5: is_real_issue = false ──┤    Decision: ACCEPT
  Worker 6: is_real_issue = true  ──┘
```

Pure vote counting with no arbiter. Each worker casts a binary vote (`is_real_issue: true/false`). The consensus percentage is calculated as `max(true_count, false_count) / total_count * 100`. Findings are accepted when consensus reaches or exceeds 60%.

**When to use:** Binary decisions where each model's vote is equally valid. Finding validation in meta-review uses majority vote because the meta-reviewers are chosen for comparable capability.

**Tradeoff:** Treats all models equally regardless of capability. A Haiku "false positive" vote counts the same as an Opus "false positive" vote. This is by design in contexts where the panel is intentionally composed of equally-strong models.

#### Weighted Vote

Each model's vote is weighted by the five-factor formula described in Section 6.6. The weighted consensus is `weighted_true / total_weight * 100`. An arbiter (default: Opus) synthesizes the weighted results, with the instruction to give higher-confidence models more influence.

**When to use:** Heterogeneous panels where model capabilities vary significantly. Code review panels that mix frontier models (Opus, GPT-4o) with smaller models (Haiku, Phi-4-mini) use weighted voting to ensure the stronger models have proportionally more influence.

**Tradeoff:** Depends on accurate weight calibration. If the tier weights or capability scores are miscalibrated, the weighted vote can be worse than majority vote. The GA model optimizer continuously refines these weights based on observed outcomes.

#### Pairwise Comparison

```
  Round 1: Opus vs Sonnet      ──┐
  Round 2: Sonnet vs Haiku     ──┼──► Arbiter synthesizes
  Round 3: Haiku vs Opus       ──┘    pairwise comparisons
```

Models review in pairs, and the arbiter (default: Opus) synthesizes the pairwise comparison results into a final decision. Each pair produces a comparison output identifying areas of agreement and disagreement.

**When to use:** Tasks where the quality of reasoning matters more than the conclusion. Architecture review uses pairwise comparison because the discussion between models often surfaces considerations that neither model would produce independently.

**Tradeoff:** Quadratic cost in the number of models. With n models, pairwise comparison requires n*(n-1)/2 pair evaluations plus an arbiter synthesis. For 6 models, this is 15 pair evaluations -- significantly more expensive than a single round of parallel queries.

### 7.3 Disagreement Detection

When workers produce conflicting responses, the consensus engine performs disagreement analysis before synthesis. The system distinguishes between three types of disagreement:

| Type | Definition | Resolution |
|------|-----------|------------|
| **Factual** | Workers disagree on a verifiable fact | Escalate to execution-based verification (OpenClaw) |
| **Interpretive** | Workers agree on facts but disagree on significance | Arbiter weighs interpretations by model capability |
| **Structural** | Workers approach the problem from incompatible frameworks | Report all perspectives, flag for human review |

Factual disagreements are the most tractable. If two models disagree about whether a function handles null inputs, this can be resolved by examining the code. The consensus engine generates a targeted follow-up query to resolve the specific point of disagreement.

Structural disagreements are the least tractable and the most valuable. When one model recommends refactoring a function and another recommends adding tests, this often indicates that the task was underspecified. The consensus engine reports both perspectives and flags the disagreement for human resolution.

### 7.4 Consensus Caching

Multi-model consensus is expensive: 4-8 model calls per consensus query, each consuming tokens and incurring latency. The consensus engine implements TTL-based caching to avoid redundant queries.

Cache keys are computed from: the prompt text (hashed), the model panel composition, and the consensus strategy. If an identical consensus query arrives within the TTL window, the cached result is returned immediately.

**Cache configuration:**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Backend | Redis | Sub-millisecond reads, automatic TTL expiry |
| Default TTL | 300 seconds | Balance between freshness and cost savings |
| Key composition | SHA-256(prompt + models + strategy) | Ensures cache hits require exact match |
| Invalidation | TTL-based only | No explicit invalidation -- stale data expires naturally |

**Tradeoff:** Caching assumes that the same prompt with the same models will produce the same consensus result. This is not strictly true (model responses are stochastic), but in practice, consensus results for the same prompt are stable within a 5-minute window. The cost savings (avoiding 4-8 API calls per cache hit) justify the marginal staleness risk.

### 7.5 Fallback Strategies

When consensus fails -- fewer than the minimum required responses arrive, or no strategy produces a decisive result -- the engine applies fallback strategies in priority order:

1. **Retry with alternative models.** Replace failing models with alternatives from the same capability tier. If Opus times out, retry with GPT-4o (same tier weight).
2. **Reduce consensus threshold.** If the normal threshold is 60%, reduce to 50% and flag the result as "low confidence."
3. **Single-model fallback.** Route to the highest-tier available model and return its response with a "no consensus" flag.
4. **Escalate to human.** For critical task types (security audit, bug detection), refuse to produce an automated result and flag for human review.

The fallback chain is traversed in order. Most failures resolve at step 1 (transient API errors). Steps 3 and 4 are rarely reached in production.

---

## 8. Multi-AI Review Pipeline

This is the system's signature pattern and its most complex workflow. It implements a four-phase adversarial review pipeline where each phase uses different models and a different arbiter. The pipeline is designed to minimize false positives through genuine adversarial pressure while maximizing true positive discovery through model diversity.

### 8.1 Pipeline Overview

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    MULTI-AI REVIEW PIPELINE                         │
  │                                                                      │
  │  Phase 1: REVIEW          Phase 2: META-REVIEW                      │
  │  ┌──────────────────┐     ┌──────────────────────┐                  │
  │  │ Opus             │     │ Fable                │                  │
  │  │ Sonnet           │     │ Hermes-405B          │                  │
  │  │ DeepSeek-Chat    │────►│ Nemotron-Ultra-550B  │                  │
  │  │ Qwen3-Coder      │     │ Qwen3-Next-80B       │                  │
  │  │                  │     │                      │                  │
  │  │ Arbiter: Opus    │     │ Arbiter: Sonnet      │                  │
  │  └──────────────────┘     └──────────┬───────────┘                  │
  │                                      │                               │
  │                           Surviving findings only                    │
  │                                      │                               │
  │  Phase 3: FIX              Phase 4: VERIFY                          │
  │  ┌──────────────────┐     ┌──────────────────────┐                  │
  │  │ Opus             │     │ Opus                 │                  │
  │  │ Sonnet           │     │ Sonnet               │                  │
  │  │ Fable            │────►│ Fable                │                  │
  │  │ Haiku            │     │                      │                  │
  │  │                  │     │ Arbiter: Haiku       │                  │
  │  │ Arbiter: Fable   │     │                      │                  │
  │  └──────────────────┘     └──────────────────────┘                  │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘
```

### 8.2 Phase 1: Review

**Objective:** Discover issues in the target artifact (code, documentation, architecture).

**Panel:** Opus, Sonnet, DeepSeek-Chat, Qwen3-Coder  
**Arbiter:** Opus  
**Execution:** All four models review the artifact concurrently. Each produces a list of findings with severity, category, file location, line hint, and description. The arbiter (Opus) synthesizes the four finding lists into a deduplicated set, resolving overlaps where multiple models identified the same issue with different descriptions.

**Why this panel composition:**
- **Opus and Sonnet** (Anthropic) provide strong general code reasoning with different capability/speed tradeoffs.
- **DeepSeek-Chat** (DeepSeek) brings a different training distribution -- it was trained heavily on code and technical content, giving it different systematic biases than the Claude family.
- **Qwen3-Coder** (Alibaba) adds a third training lineage. Its code understanding reflects different optimization objectives and data mixtures.

The four models span three organizations (Anthropic, DeepSeek, Alibaba), ensuring that correlated training biases are minimized. A bug that Claude's training data taught it to ignore may be caught by DeepSeek or Qwen, and vice versa.

**Confidence threshold:** Only findings with confidence >= 70% are forwarded to meta-review. This pre-filter reduces the meta-review workload without discarding borderline findings (the 70% threshold is set conservatively -- it is better to send a questionable finding to meta-review than to silently discard it).

### 8.3 Phase 2: Meta-Review

**Objective:** Adversarially challenge every finding from Phase 1. Default posture is skepticism.

**Panel:** Fable, Hermes-3-LLaMA-3.1-405B, Nemotron-Ultra-550B, Qwen3-Next-80B  
**Arbiter:** Sonnet  
**Execution:** Each finding is independently reviewed by all four meta-review models. Each model receives the finding details and the relevant source code, with explicit instructions to attempt refutation.

**The meta-review prompt (core instruction):**

```
You are an ADVERSARIAL reviewer. Your job is to CHALLENGE this finding.
Try to REFUTE it. Default to skepticism -- only confirm if you cannot
find a reason to reject.

...

Be HARSH. Better to reject a valid finding than to waste effort fixing
a non-issue.
```

Each meta-reviewer produces a verdict:

| Verdict | Meaning |
|---------|---------|
| `confirmed` | Definitely a real issue -- meta-reviewer could not refute it |
| `likely_valid` | Probably real, but with some uncertainty |
| `questionable` | Uncertain -- could go either way |
| `false_positive` | Not a real issue -- meta-reviewer found a reason to reject |

**Survival rule:** A finding survives meta-review only if `confirmed + likely_valid > false_positive + questionable`. This is a strict majority vote. A finding that receives 2 confirmations and 2 false-positive verdicts does not survive -- the tie breaks toward rejection, consistent with the skeptical default.

**Why zero overlap with the review panel:**

This is the most important architectural decision in the pipeline. Consider what happens with overlap:

1. Opus identifies a potential null-pointer dereference (Phase 1, Review).
2. Opus evaluates whether its own finding is valid (Phase 2, Meta-Review).

In step 2, Opus will almost certainly confirm its own finding. The model's weights encode the same systematic biases that produced the finding in the first place. This is not independent review -- it is self-confirmation.

With zero overlap:

1. Opus identifies a potential null-pointer dereference (Phase 1, Review).
2. Hermes-405B evaluates whether Opus's finding is valid (Phase 2, Meta-Review).

Hermes-405B was trained by NousResearch on a different data mixture with different optimization objectives. Its systematic biases are uncorrelated with Opus's. If Hermes-405B confirms the finding, this provides genuine independent evidence. If it refutes the finding, this is a genuine adversarial challenge that Opus's self-review could never produce.

The cost of zero overlap is that the meta-review panel cannot include the system's strongest code reasoner (Opus) or its most balanced reviewer (Sonnet). The system compensates by selecting meta-reviewers of comparable capability: Hermes-405B (405B parameters), Nemotron-Ultra-550B (550B parameters), and Qwen3-Next-80B are all frontier-class models that can meaningfully challenge Opus-tier findings.

### 8.4 Phase 3: Fix

**Objective:** Generate fix proposals for findings that survived meta-review.

**Panel:** Opus, Sonnet, Fable, Haiku  
**Arbiter:** Fable  
**Execution:** Each model independently generates a fix for each surviving finding. The arbiter (Fable) evaluates all fix proposals and selects the best one based on correctness, minimal invasiveness, and consistency with the codebase's style.

**Why Fable as fix arbiter:** Fable provides near-Opus quality with a different evaluation perspective. Using Opus as both a fix generator and the fix arbiter would introduce the same self-confirmation bias that zero overlap prevents in the review/meta-review phases. By making Fable the arbiter for fix selection, no model both generates and judges its own fix.

**Fix application:** The selected fix is applied to the working tree using standard git operations. If the fix involves multiple files, all changes are applied atomically. The workflow tracks which findings were addressed by which fixes for traceability.

### 8.5 Phase 4: Verify

**Objective:** Confirm that applied fixes do not introduce regressions.

**Panel:** Opus, Sonnet, Fable  
**Arbiter:** Haiku  
**Execution:** The verification panel reviews the diff produced by Phase 3, comparing the before and after states. Each model checks for: introduced bugs, broken edge cases, style violations, and consistency with the original fix intent.

**Why Haiku as verify arbiter:** Verification is a binary decision (pass/fail) on a narrower scope (a diff, not the full codebase). Haiku's speed advantage (5-10x faster than Opus) makes it cost-effective for this simpler synthesis task. Using a stronger arbiter here would add cost without proportional quality improvement, because the verification task is inherently less complex than the review or fix-selection tasks.

### 8.6 Four Different Arbiters

Each phase uses a different arbiter model. This is not arbitrary -- it is a structural defense against arbiter bias.

| Phase | Arbiter | Rationale |
|-------|---------|-----------|
| Review | Opus | Strongest synthesizer for complex, multi-model finding deduplication |
| Meta-Review | Sonnet | Balanced evaluator for adversarial verdict aggregation; different from Review arbiter |
| Fix | Fable | Near-Opus quality; prevents Opus from judging its own fix proposals |
| Verify | Haiku | Fast, cost-effective for binary pass/fail on narrow diffs |

If the same arbiter were used for all phases, its systematic biases would compound across the pipeline. An arbiter that tends to accept findings (Phase 1) would also tend to confirm those findings in meta-review (Phase 2), accept fixes for them (Phase 3), and verify those fixes (Phase 4). Using four different arbiters introduces four independent bias profiles, preventing any single model's preferences from dominating the pipeline end-to-end.

### 8.7 Re-Review: Rotated Panel Recurrence

When the fix verification rate falls below 80% (indicating that fixes are frequently introducing regressions), the orchestrator triggers a re-review cycle. The re-review uses **rotated panels** -- the review and meta-review models are shuffled into different positions:

```
  Standard Pipeline:
    Review:      [Opus, Sonnet, DeepSeek, Qwen3-Coder]
    Meta-Review: [Fable, Hermes-405B, Nemotron-550B, Qwen3-Next-80B]

  Re-Review (rotated):
    Review:      [Fable, Hermes-405B, Nemotron-550B, Qwen3-Next-80B]
    Meta-Review: [Opus, Sonnet, DeepSeek, Qwen3-Coder]
```

The models that were meta-reviewers become reviewers, and vice versa. This rotation ensures that the re-review is not simply re-running the same analysis. Models that previously challenged findings now discover them, and models that previously discovered findings now challenge them. The zero-overlap invariant is maintained across the rotation.

### 8.8 External Adversarial Layer

The internal pipeline (Phases 1-4) operates within a single ecosystem of models accessed via API. While the models span multiple providers, they are all invoked by the same orchestrator, evaluated by the same consensus engine, and their results are stored in the same database. This creates a subtle risk: **feedback collapse**.

Feedback collapse occurs when the system's routing optimizations begin favoring models whose outputs align with the consensus, gradually narrowing the distribution of model responses. Over time, the ensemble converges toward a single perspective, losing the diversity that justifies multi-model review in the first place.

To counter this, the system incorporates external adversarial review from models outside the orchestration loop:

**ChatGPT (co-architect role):**
- Designed the two-layer evaluation architecture.
- Performs periodic "truth audits" of the system's outputs and metrics.
- Operating principle: "Assume system is wrong until proven otherwise."
- Evaluates on five dimensions: Correctness, Robustness, Generalization, Bias Resistance, Reproducibility.
- Findings from ChatGPT audits are opened as issues that the fleet addresses through the standard pipeline.

**Grok (brutal honesty role):**
- Used for external research verification where unfiltered assessment is valuable.
- Grok's training on real-time data (via X/Twitter) provides a different information baseline than models trained on static corpora.
- Deployed selectively for fact-checking and pricing verification tasks.

**External review process:**

```
  Internal Pipeline                    External Review
  ─────────────────                    ───────────────
  Review → Meta-Review                 ChatGPT truth audit
  → Fix → Verify                       Grok verification
       │                                    │
       │   Periodic export                  │
       │   of outputs + metrics             │
       └──────────────────────────────────►│
                                            │
                                   Independent evaluation
                                   (5-dimension scoring)
                                            │
                                   Issues opened for
                                   findings that fail
                                   external review
                                            │
       ┌◄───────────────────────────────────┘
       │
  Fleet addresses issues
  through standard pipeline
```

**Why external matters:** The internal pipeline can detect issues that individual models miss, but it cannot detect biases that all models in the pool share. For example, if every model in the pipeline was trained on the same flawed documentation about a library's API, they will all produce the same incorrect understanding, and the consensus engine will report high confidence in that incorrect understanding. An external model trained on different data may not share this blind spot.

The feedback loop optimizer (`tools/feedback_loop_optimizer.py`) monitors four risk indicators:

1. **Model dominance:** One model receives >70% of routing traffic (indicates selection bias).
2. **Evaluator-generator coupling:** A model evaluates its own outputs >40% of the time (indicates self-confirmation).
3. **Reward hacking:** Quality metrics increase while diversity metrics decrease (indicates metric overfitting).
4. **Concept collapse:** Output embeddings for different queries converge beyond 0.90 similarity (indicates response homogenization).

When any indicator exceeds its threshold, the system generates alerts stored in `monitoring.diversity_alerts` and triggers corrective actions (forced model rotation, panel reshuffling, or routing weight reset).

### 8.9 Complete Pipeline Sequence Diagram

```
Client          Orchestrator      Review Panel       Meta-Review Panel    Fix Panel      Verify Panel
  │                  │                │                     │                │               │
  │ POST /review     │                │                     │                │               │
  │─────────────────►│                │                     │                │               │
  │                  │                │                     │                │               │
  │                  │  ═══════════════════════════════════════════════════════════════════  │
  │                  │  ║ PHASE 1: REVIEW (Arbiter: Opus)                               ║  │
  │                  │  ═══════════════════════════════════════════════════════════════════  │
  │                  │                │                     │                │               │
  │                  │── review ─────►│ Opus                │                │               │
  │                  │── review ─────►│ Sonnet              │                │               │
  │                  │── review ─────►│ DeepSeek-Chat       │                │               │
  │                  │── review ─────►│ Qwen3-Coder         │                │               │
  │                  │                │   (concurrent)       │                │               │
  │                  │◄─ findings ───│                     │                │               │
  │                  │                │                     │                │               │
  │                  │  Opus arbiter deduplicates           │                │               │
  │                  │  Filter: confidence >= 70%           │                │               │
  │                  │                │                     │                │               │
  │                  │  ═══════════════════════════════════════════════════════════════════  │
  │                  │  ║ PHASE 2: META-REVIEW (Arbiter: Sonnet)                        ║  │
  │                  │  ═══════════════════════════════════════════════════════════════════  │
  │                  │                │                     │                │               │
  │                  │  For each finding:                   │                │               │
  │                  │── challenge ──────────────────────►│ Fable           │               │
  │                  │── challenge ──────────────────────►│ Hermes-405B     │               │
  │                  │── challenge ──────────────────────►│ Nemotron-550B   │               │
  │                  │── challenge ──────────────────────►│ Qwen3-Next-80B  │               │
  │                  │                                     │   (concurrent)  │               │
  │                  │◄─ verdicts ─────────────────────────│                │               │
  │                  │                                     │                │               │
  │                  │  Majority vote: confirmed+likely > false_pos+quest   │               │
  │                  │  Surviving findings only continue    │                │               │
  │                  │                │                     │                │               │
  │                  │  ═══════════════════════════════════════════════════════════════════  │
  │                  │  ║ PHASE 3: FIX (Arbiter: Fable)                                 ║  │
  │                  │  ═══════════════════════════════════════════════════════════════════  │
  │                  │                │                     │                │               │
  │                  │  For each surviving finding:         │                │               │
  │                  │── fix ────────────────────────────────────────────►│ Opus           │
  │                  │── fix ────────────────────────────────────────────►│ Sonnet         │
  │                  │── fix ────────────────────────────────────────────►│ Fable          │
  │                  │── fix ────────────────────────────────────────────►│ Haiku          │
  │                  │                                                    │   (concurrent) │
  │                  │◄─ fix proposals ──────────────────────────────────│               │
  │                  │                                                    │               │
  │                  │  Fable arbiter selects best fix                    │               │
  │                  │  Apply to working tree                             │               │
  │                  │                │                     │                │               │
  │                  │  ═══════════════════════════════════════════════════════════════════  │
  │                  │  ║ PHASE 4: VERIFY (Arbiter: Haiku)                              ║  │
  │                  │  ═══════════════════════════════════════════════════════════════════  │
  │                  │                │                     │                │               │
  │                  │── verify diff ──────────────────────────────────────────────────►│  │
  │                  │                                                                  │  │
  │                  │◄─ pass/fail ─────────────────────────────────────────────────────│  │
  │                  │                │                     │                │               │
  │                  │  Haiku arbiter: binary pass/fail     │                │               │
  │                  │                │                     │                │               │
  │  {findings,      │                │                     │                │               │
  │   false_pos_rate,│                │                     │                │               │
  │   fix_rate,      │                │                     │                │               │
  │   verify_rate}   │                │                     │                │               │
  │◄─────────────────│                │                     │                │               │
  │                  │                │                     │                │               │
```

### 8.10 Pipeline Metrics

The pipeline tracks the following metrics across runs, stored in `data/code-review-and-solve.json` and `monitoring.execution_summary`:

| Metric | Definition | Target |
|--------|-----------|--------|
| Findings discovered | Raw count from Phase 1 | Maximize |
| Meta-review survival rate | Findings surviving Phase 2 / total findings | 40-70% |
| False positive rate | 1 - survival rate | < 60% |
| Fix success rate | Fixes applied successfully / surviving findings | > 80% |
| Verification pass rate | Fixes passing Phase 4 / fixes applied | > 80% |
| Total cost | Sum of API token costs across all phases | Minimize |
| Total latency | Wall-clock time from request to final result | < 10 min for typical repos |

A meta-review survival rate below 40% suggests the review panel is too aggressive (producing many low-quality findings). A survival rate above 70% suggests the meta-review panel is too lenient (not challenging findings effectively). The GA team selector evolves panel composition to maintain survival rates in the target range.

### 8.11 Design Alternatives Considered

**Alternative 1: Single-model review with self-critique.**  
Approach: Ask one model (e.g., Opus) to review code, then ask it to critique its own findings.  
Rejection reason: Self-critique does not introduce new perspectives. The model's posterior distribution does not meaningfully shift when asked to "double-check." Empirically, self-critique eliminates < 5% of false positives.

**Alternative 2: Same-panel meta-review.**  
Approach: Use the same models for review and meta-review (e.g., Opus reviews, then Opus meta-reviews).  
Rejection reason: Same models share the same training biases. Correlated errors provide no ensemble benefit.

**Alternative 3: Human-in-the-loop meta-review.**  
Approach: Send all findings to a human reviewer for validation before fixing.  
Rejection reason: Defeats the purpose of automation. The pipeline is designed to run unattended. Human review is preserved as the final validation layer (after Phase 4), but the pipeline should produce results that are good enough to merge without human intervention in the common case.

**Alternative 4: Execution-based verification only.**  
Approach: Skip model-based review entirely. Run the code, run the tests, trust the results.  
Rejection reason: Many code quality issues (security vulnerabilities, race conditions, architectural debt) are not caught by test suites. Model-based review catches issues that execution cannot, while execution catches issues that models cannot. The pipeline supports both (OpenClaw provides execution-backed verification as an optional 7th worker).

---

## 9. Knowledge Pipeline

The knowledge pipeline is a 4-stage asynchronous system that transforms raw web content
into searchable, graph-connected knowledge. Every stage is decoupled through Redis sorted
sets, and every stage communicates exclusively through the REST API -- workers never touch
the filesystem or database directly.

```
                                    Redis Sorted Sets
                                    (priority-ordered)

  +----------+     +-------+     +---------+     +---------+     +---------+
  | Scrapers |---->| store |---->|  chunk  |---->|  embed  |---->|  graph  |
  | (88+)    |     | queue |     |  queue  |     |  queue  |     |  queue  |
  +----------+     +---+---+     +----+----+     +----+----+     +----+----+
                       |              |               |               |
                  +----v----+   +----v----+     +----v----+     +----v----+
                  | Store   |   | Chunk   |     | Embed   |     | Graph   |
                  | Worker  |   | Worker  |     | Worker  |     | Worker  |
                  +---------+   +---------+     +---------+     +---------+
                       |              |               |               |
                       v              v               v               v
                  PostgreSQL     PostgreSQL       pgvector        OrientDB
                  (documents)    (chunks)        (embeddings)     (vertices
                                                                  + edges)
```

**Why this architecture exists.** The original design ran scraping and embedding
synchronously on the same node. A single scraper would fetch a page, chunk it, embed it,
and store it -- all in one blocking loop. This created two problems: embedding (the
compute-intensive step) throttled scraping (the I/O-bound step), and a single failure in
any stage killed the entire pipeline for that document. Decoupling through Redis queues
solved both: scrapers run at network speed while embedding runs at GPU speed, and each
stage can fail and retry independently.

**Performance impact.** Async decoupling improved throughput from 599 docs/hour to
4,700 docs/hour -- a 7.8x improvement with 13 active scrapers. The bottleneck shifted
from "embedding blocks scraping" to "how many scrapers can we run in parallel."

**Pipeline chaining.** Stages chain automatically. When a store worker calls
`POST /queue/complete`, the REST API consults a static map and enqueues the item to the
next stage:

```python
PIPELINE_ORDER = ('store', 'chunk', 'embed', 'graph')
PIPELINE_NEXT  = {'store': 'chunk', 'chunk': 'embed', 'embed': 'graph'}
```

No worker needs to know what comes after it. Adding a new stage (e.g., a summarization
stage between chunk and embed) requires only updating this map.


### 9.1 Web Scraping

The scraping layer consists of 88+ scraper implementations across 12 knowledge domains,
all derived from a single `BaseScraper` class.

**Domain coverage:**

| Domain | Scrapers | Example Sources |
|--------|----------|-----------------|
| Programming Languages | 14 | Python docs, Rust book, Go stdlib |
| DevOps / Cloud | 15 | Kubernetes, Terraform, AWS docs |
| Web Frameworks | 6 | React, Django, FastAPI |
| Databases | 5 | PostgreSQL, Redis, MongoDB |
| Linux / OS | 8 | Kernel docs, systemd, man pages |
| Security | 3 | OWASP, CVE databases |
| Electronics / FPGA | 8 | Verilog, VHDL, chip datasheets |
| Science | 6 | ArXiv abstracts, PubMed |
| Mathematics | 3 | Wolfram, NIST standards |
| Food / Cooking | 6 | Recipe databases |
| ML / Data | 4 | PyTorch, scikit-learn |
| Wikipedia (deep) | 45 topics | Depth-3 recursive, up to 2000 pages/topic |

**BaseScraper pattern.** Every scraper subclasses `BaseScraper` and overrides a single
method:

```python
class BaseScraper:
    def __init__(self, source_name, base_dir, interval_seconds=900):
        self.output_dir = f"{base_dir}/scraped-data/{source_name}/"
        # ...

    def scrape(self):
        """Override this. Yield items to be stored."""
        raise NotImplementedError

    def make_id(self, content):
        """SHA-256 truncated to 16 hex chars for deduplication."""
        return hashlib.sha256(content.encode()).hexdigest()[:16]

    def save_item(self, item):
        """Batches items (groups of 20), then enqueues via REST API."""
        self._batch.append(item)
        if len(self._batch) >= 20:
            self._flush_batch()  # POST http://aio-01:5000/queue/enqueue
```

A new scraper requires only defining `SOURCES` (a dict of URLs) and implementing
`scrape()`. The base class handles batching, deduplication, rate limiting, error recovery,
and REST API communication.

**Design decisions.**

- *Content truncation at 50,000 characters.* Pages larger than this are typically
  auto-generated API references or data dumps that produce low-quality embeddings. The
  minimum threshold of 50 characters filters out error pages and empty responses.

- *Batch size of 20.* Balances HTTP overhead (fewer requests) against memory pressure
  (items buffered in-process). With an average item size of 10-15KB, each batch is
  roughly 200-300KB -- small enough that a worker crash loses at most 20 items.

- *Rate limiting per scraper.* Each scraper specifies its own delay (1.0-2.0 seconds
  between requests). Kernel documentation uses 1.5s; Red Hat and Wikipedia use 2.0s. The
  `User-Agent` header identifies the bot as `ResearchScraper/1.0 (academic research)`.

**Fleet distribution.** Scrapers deploy to fleet workers via an Ansible role that places
them at `/home/claude/workers/scrapers/` on each node. The orchestrator dispatches
scraping tasks via `POST /fleet/deploy`, distributing work across available workers based
on current load.

**Alternatives considered.** We evaluated Scrapy (too heavyweight for simple page
fetching, adds dependency management overhead), Playwright (needed only for JS-rendered
pages, which are a minority of our sources), and direct `requests` (no built-in rate
limiting or retry). The `urllib.request` stdlib approach with the BaseScraper wrapper
provides the right abstraction level: simple enough to understand in 5 minutes,
extensible enough to handle 88+ domains.

**Limitations.** The current scraping layer does not handle JavaScript-rendered content.
Single-page applications (SPAs) and sites that require authentication are not supported.
There is no central scraper registry -- adding a new scraper requires deploying a new
Python file to fleet workers.


### 9.2 Store Stage

The store stage is the entry point for all scraped content into the persistent pipeline.
Its job is deduplication, persistence, and automatic chaining to the chunk stage.

**Flow:**

```
Scraper                      REST API                    Redis
  |                            |                          |
  |-- POST /queue/enqueue ---->|                          |
  |   {queue:"store",          |-- ZADD pipeline:store:   |
  |    items:[...],            |   pending (score, id) -->|
  |    priority:5}             |                          |
  |                            |                          |
  Store Worker                 |                          |
  |                            |                          |
  |-- POST /queue/fetch/store->|                          |
  |                            |<-- ZPOPMIN (atomic) -----|
  |<-- item payload -----------|                          |
  |                            |                          |
  |-- POST /store ------------>|                          |
  |   {content, metadata}      |-- INSERT INTO            |
  |                            |   knowledge.documents -->| PostgreSQL
  |                            |                          |
  |-- POST /queue/complete --->|                          |
  |   {item_id, queue:"store"} |-- ZADD pipeline:chunk:   |
  |                            |   pending (score, id) -->| (auto-chain)
```

**Deduplication.** The REST API maintains idempotency keys in Redis with a 24-hour TTL.
When a scraper enqueues an item with an idempotency key that already exists, the request
succeeds silently (HTTP 200) but the item is not added to the queue. Content-based
deduplication uses SHA-256 hashes stored in `dedup.content_hashes`.

**Why workers never write to the filesystem.** In early iterations, store workers wrote
scraped content to NFS-mounted directories. This created three problems: NFS latency
spikes during heavy writes, permission issues across heterogeneous fleet nodes, and no
transactional guarantee that the file write and database insert would both succeed. Moving
all persistence behind the REST API eliminated all three.

**Priority scoring.** Items enter the pending sorted set with a composite score:

```
score = (10 - priority) * 10,000,000,000,000 + timestamp_ms
```

Lower scores sort first (Redis `ZPOPMIN`), so higher-priority items (priority 10) get
score band 0-9,999,999,999,999, while default-priority items (priority 5) get score band
50,000,000,000,000-59,999,999,999,999. Within the same priority band, earlier timestamps
sort first, preserving FIFO order.

**Tradeoff.** This scoring scheme sacrifices human-readable scores for a single-field
encoding of two dimensions (priority + time). An alternative would be two separate sorted
sets (one per priority level), but that would require workers to check multiple queues and
implement priority logic themselves.


### 9.3 Chunking

Chunking splits stored documents into focused, retrievable units sized for LLM context
windows and embedding models.

**Primary chunker configuration:**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Min chunk size | 500 chars | Below this, chunks lack sufficient context for meaningful embeddings |
| Max chunk size | 1,500 chars | Above this, embedding precision degrades (single vector represents too many concepts) |
| Overlap | 100 chars | Preserves continuity across chunk boundaries |
| Max text size | 10 MB | Memory safety limit for the chunking process |

**Splitting strategy.** The chunker uses a two-phase approach:

1. *Content classification.* Heuristic pattern matching detects code blocks (Python,
   JavaScript, Java, SQL, Bash) via function/class definition patterns (`def `, `class `,
   `function `, `public `, `private `).

2. *Boundary-aware splitting.* Prose splits on paragraph boundaries (double newlines) or
   markdown headers. Code splits on function/class boundaries. Neither ever splits
   mid-sentence or mid-statement. If a single logical unit exceeds `max_chunk_size`, a
   force-split occurs at the nearest whitespace boundary.

Each chunk carries metadata: `index`, `content`, `overlap_prefix`, `overlap_suffix`,
`char_count`, `has_code`, `language`, `chunk_type`. The overlap fields enable downstream
consumers to reconstruct context across chunk boundaries when needed.

**Why chunking matters.** Embedding models map text to fixed-dimensional vectors. A single
vector for a 50,000-character document would need to encode hundreds of distinct concepts,
diluting the representation of each. Chunking ensures each vector represents a focused
topic, improving retrieval precision.

**The fundamental tradeoff.** Larger chunks preserve more context (a full function with its
docstring) but reduce retrieval precision (the vector is a blurred average of all
concepts in the chunk). Smaller chunks improve precision (each vector maps to one concept)
but lose context (a code snippet without its surrounding explanation). The 500-1,500
character range represents our empirical sweet spot for mixed prose-and-code technical
documentation.

**Known inconsistency.** Different entry points use different chunk sizes: the primary
`SemanticChunker` uses 500-1,500 chars; the pipeline worker uses 4,096 chars with 1,500
overlap; standalone ingestion uses 1,000 chars with no overlap. This inconsistency means
chunks from different ingestion paths have different granularity, which can affect
retrieval quality. Standardizing on a single configuration is a known improvement target.

**Alternatives considered.** Fixed-size splitting (simple but destroys semantic
boundaries), recursive character splitting (LangChain-style -- better but still
syntax-unaware), and AST-based splitting for code (precise but language-specific and
brittle). The heuristic approach balances generality with quality.


### 9.4 Embedding

The embedding stage transforms text chunks into 768-dimensional vectors for similarity
search.

**Primary model:** `sentence-transformers/all-mpnet-base-v2` -- a general-purpose sentence
embedding model that produces 768-dimensional vectors. Selected for its balance of quality
(top-tier on MTEB benchmarks at its size class) and inference speed on CPU hardware.

**Embedding models in use:**

| Model | Dimensions | Provider | Role |
|-------|-----------|----------|------|
| `all-mpnet-base-v2` | 768 | Local (sentence-transformers) | Primary embedding |
| `bge-large-en-v1.5` | 1024 | Cloudflare Workers AI | Proxy fallback tier 1 |
| `bge-base-en-v1.5` | 768 | Cloudflare Workers AI | Proxy fallback tier 2 |
| `text-embedding-004` | 768 | Google | Proxy fallback tier 3 |
| `ms-marco-MiniLM-L-6-v2` | N/A | Local cross-encoder | Re-ranking only (not embedding) |

**Compute isolation constraint.** Embedding inference runs exclusively on dedicated
compute nodes (laptop-01 and laptop-02), never on fleet workers. This is an intentional
architectural constraint, not a limitation.

*Why this matters:* Fleet workers are lightweight machines (1-8GB RAM) optimized for
I/O-bound orchestration tasks -- HTTP requests, SSH commands, queue management. Loading a
sentence-transformers model consumes 400-800MB of RAM and saturates CPU during inference.
Running embeddings on a 1GB fleet worker would either OOM-kill the process or starve other
orchestration tasks of resources. Dedicated compute nodes have 31GB RAM and can run
multiple embedding workers concurrently without resource contention.

**Cascading fallback.** Query-time embedding generation tries multiple endpoints in
sequence:

```python
EMBED_ENDPOINTS = [
    "http://server-03:5001/embed",              # Primary: dedicated embed service
    "http://localhost:5000/learning/embeddings/generate",  # Fallback: local API
    "http://localhost:5001/embed"                # Last resort: local standalone
]
```

If all HTTP endpoints fail, a hash-based pseudo-embedding fallback generates a
deterministic 768-dimensional vector from TF-IDF-like token hashing using SHA-256. This
produces geometrically meaningful (though lower quality) vectors that allow the pipeline
to continue rather than halt.

**HNSW indexing.** Vectors are indexed using PostgreSQL pgvector's HNSW algorithm:

```sql
CREATE INDEX idx_chunks_embedding ON documents.chunks
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
```

HNSW parameters: `m = 16` (connections per node in the graph -- balances index size
against recall quality), `ef_construction = 64` (build-time search width -- higher values
improve recall at the cost of slower index builds). Query-time `ef_search` uses the
pgvector default of 40.

**IVFFlat for large tables.** The `knowledge.embeddings` table (700K+ vectors) uses
IVFFlat indexing (`lists = 100`) instead of HNSW. HNSW would require 2GB+ of
`maintenance_work_mem` and 30+ minutes to build on the available hardware. IVFFlat builds
in seconds with a modest memory footprint at the cost of slightly lower recall (requires
probing multiple cluster centroids).

**Tradeoffs.** 768 dimensions is the standard, but the codebase has dimension
inconsistencies: workflow task embeddings use 384 dimensions, and `bge-large-en-v1.5`
produces 1024 dimensions. Vectors from different models cannot be compared directly --
the system handles this by segregating vectors into model-specific columns and tables.


### 9.5 Graph Sync

The final pipeline stage creates graph relationships in OrientDB, enabling multi-hop
traversal queries that would be prohibitively expensive in SQL.

**Flow for each completed embedding:**

1. Claim item from `pipeline:graph:pending` (Redis sorted set).
2. Create a `Document` vertex in OrientDB with `chunk_id`, `category`, `url_hash`.
3. Query PostgreSQL for the top-10 most similar documents via pgvector cosine similarity.
4. Create `RelatedTo` edges between the new vertex and each similar document, with
   similarity scores as edge properties.

**Triple-store architecture.** Knowledge lives across three databases, each handling what
it does best:

```
+------------------+     +------------------+     +------------------+
|   PostgreSQL     |     |     Redis        |     |   OrientDB       |
|   + pgvector     |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
| Structured data  |     | Pipeline queues  |     | Relationships    |
| Vector similarity|     | Priority scoring |     | Multi-hop queries|
| Full-text search |     | Idempotency keys |     | Shortest path    |
| ACID guarantees  |     | Worker heartbeats|     | Bidirectional    |
| SQL analytics    |     | In-memory speed  |     | Graph traversal  |
+------------------+     +------------------+     +------------------+
```

**Why a separate graph database.** Consider the query: "Find all documents related to
Document A through at most 3 intermediate documents." In SQL, this requires 3 self-joins
on a relationship table, each potentially scanning thousands of rows. With 315K+
documents, the query plan becomes exponentially expensive with each hop. In OrientDB, this
is a single `TRAVERSE` command that follows edges directly, executing in under 10ms
regardless of dataset size.

**Fault tolerance.** Graph sync is non-blocking and best-effort. Failures in OrientDB
operations are logged but never fail the parent pipeline operation. The rationale: graph
relationships are a performance optimization for traversal queries, not a source of truth.
PostgreSQL holds the canonical data; OrientDB can always be rebuilt from it via
`POST /api/graph/load-from-postgres`.

**Limitation.** Two stale Neo4j client files remain in the codebase (`shared/workflow-graph-sync.js`,
`shared/neo4j-realtime-sync.cjs`). They import `neo4j-driver` and silently fail at
runtime since Neo4j does not run anywhere. These are technical debt awaiting cleanup.

---

## 10. Vector Search

Vector search is the primary retrieval mechanism for the knowledge base. It combines
four layers to maximize both recall and precision.

**Layer architecture:**

```
  User Query
      |
      v
+-----+------+          +------+------+
| Vector      |          | Full-Text   |
| Search      |          | Search      |
| (pgvector)  |          | (tsvector)  |
+------+------+          +------+------+
       |                        |
       |   Top-N results each   |
       |                        |
       +----------+-------------+
                  |
                  v
        +---------+---------+
        | Reciprocal Rank   |
        | Fusion (K=60)     |
        +---------+---------+
                  |
                  v
        +---------+---------+
        | Cross-Encoder     |
        | Re-ranking        |
        | (optional)        |
        +---------+---------+
                  |
                  v
           Final Results
```

**Layer 1: Vector similarity.** pgvector's cosine distance operator (`<=>`) computes
similarity between the query embedding and all indexed document embeddings. HNSW indexing
reduces this from O(n) brute-force to O(log n) approximate nearest neighbor.

```sql
SELECT c.id, c.content,
       1 - (e.embedding <=> query_vector::vector) AS similarity
FROM knowledge.embeddings e
JOIN knowledge.chunks c ON e.chunk_id = c.id
ORDER BY e.embedding <=> query_vector::vector
LIMIT 20;
```

Query latency: 0.4ms average on the 700K+ vector table with IVFFlat indexing. This is
2-6x faster than ChromaDB on equivalent workloads, primarily because pgvector avoids the
Python-to-C++ serialization overhead that ChromaDB incurs on every query.

| Operation | ChromaDB | pgvector | Speedup |
|-----------|----------|----------|---------|
| Simple similarity (128-dim) | 0.9ms | 0.44ms | 2.0x |
| Filtered similarity | 2.3ms | 0.44ms | 5.2x |
| 768-dim vectors | 1.2ms | 0.97ms | 1.2x |

**Layer 2: Full-text search.** PostgreSQL's built-in `tsvector`/`tsquery` provides
keyword-based retrieval that catches exact matches that semantic search might miss (e.g.,
specific function names, error codes, configuration keys).

```sql
SELECT id, content,
       ts_rank_cd(tsv, plainto_tsquery('english', query)) AS text_rank
FROM knowledge.chunks
WHERE tsv @@ plainto_tsquery('english', query)
ORDER BY text_rank DESC
LIMIT 20;
```

The `tsv` column is auto-populated by a PostgreSQL trigger on insert/update, so there is
no application-level indexing step.

**Layer 3: Reciprocal Rank Fusion (RRF).** RRF merges the two result sets using a
rank-based scoring formula that is agnostic to the underlying score distributions:

```
RRF_score(d) = sum over all rankings R of: 1 / (K + rank_R(d) + 1)
```

With `K = 60` (the standard constant from the original RRF paper). Documents appearing in
both result sets receive additive score contributions from each, naturally boosting items
that are both semantically similar and keyword-relevant.

**Why RRF over learned fusion.** Score normalization (scaling vector similarity and text
rank to a common range) requires calibration data and is sensitive to score distribution
changes. RRF uses only rank positions, making it robust to score scale changes across
different queries and index sizes. The tradeoff: RRF cannot distinguish between a document
ranked #1 with 0.99 similarity and one ranked #1 with 0.51 similarity -- both get the
same RRF contribution.

**Layer 4: Cross-encoder re-ranking (optional).** The `ms-marco-MiniLM-L-6-v2`
cross-encoder runs as a separate Flask microservice on port 5002, restricted to
laptop-01/laptop-02 (the same compute isolation constraint as embedding). When re-ranking
is enabled, the system over-fetches 5x from layers 1-3, then re-ranks the full set using
the cross-encoder's pairwise relevance scores.

Cross-encoders produce higher-quality relevance judgments than bi-encoders (they see the
query and document together, not as independent embeddings), but they are O(n) per query
rather than O(1) per pre-computed embedding. This is why they are used only for re-ranking
a small candidate set, not for initial retrieval.

**Meta-layer: Cross-source fusion.** A second RRF pass in `intelligent_search.py`
merges results from four sources: the knowledge base, learning memories, learning
experiences, and OrientDB graph traversals. This enables queries like "how did we solve X
last time?" to surface both documentation and past workflow results.

---

## 11. Database Architecture

The system uses three databases, each chosen for a specific access pattern. This is not
accidental complexity -- it is intentional specialization.


### 11.1 PostgreSQL + pgvector

PostgreSQL serves as the system of record for all structured data, vector storage, and
full-text search. It runs on aio-01 (port 5433) as PostgreSQL 17.9 with the pgvector
extension.

**Scale:** 25 schemas, 226 tables, 50+ vector indexes.

**Schema organization:**

| Schema | Tables | Purpose |
|--------|--------|---------|
| `knowledge` | documents, chunks, embeddings, concepts, code_embeddings, web_articles, scraped_content, research_findings | Knowledge base (700K+ chunks) |
| `learning` | strategy_performance, experiences, model_capabilities, memories, icl_examples, bandit_models | Model learning and strategy selection |
| `workflow` | executions, worker_results, arbiter_decisions, phases, feedback, learnings, consensus_cache, confidence_calibration | Multi-AI workflow orchestration |
| `monitoring` | execution_summary, api_health_status, rate_limits, fallback_attempts, prediction_accuracy | Execution monitoring and alerting |
| `ga` | populations, best_solutions, convergence_metrics, migration_log | Genetic algorithm state |
| `experiments` | registry, runs | A/B testing |
| `costs` | entries | Cost tracking |
| `fleet` | workers, tasks, workflows, command_results | Fleet management |
| `documents` | documents, chunks, processing_log | Document ingestion pipeline |
| `search` | knowledge_base | Full-text search (generated tsvector) |
| `facts` | facts | SPO triple extraction |
| `reasoning` | hypotheses, evidence, explanations | Hypothesis tracking |
| `auth` | api_keys, secrets | API key management |
| `scraping` | tasks | Web scraping pipeline |
| `dedup` | content_hashes, similar_pairs | Content deduplication |
| `inventory` | machines | Infrastructure inventory |
| `orchestration` | task_queue, auto_storage, conversation_history | Task orchestration |
| Others (9) | Various | admin, config, queue, processing, orchestrator, storage, auto_storage, workflows (legacy), admin |

**Vector dimensions and indexing strategy:**

| Dimension | Index Type | Tables | Rationale |
|-----------|-----------|--------|-----------|
| 768 | HNSW (m=16, ef=64) | documents.chunks, learning.*, most knowledge.* | Standard for <100K rows; best recall |
| 768 | IVFFlat (lists=100) | knowledge.embeddings (700K+), workflow.worker_results | Large tables; faster build, modest recall tradeoff |
| 384 | IVFFlat | workflow.executions.task_embedding | Lighter-weight task similarity |
| 128 | Legacy | learning.experiences (migrating to 768) | Historical; `embedding_old_128` column preserved |

**Why HNSW vs IVFFlat.** HNSW provides better recall (~99% vs ~95% for IVFFlat with
`lists=100`) and does not require a separate training step. However, building an HNSW
index on 700K+ vectors requires 2GB+ `maintenance_work_mem` and 30+ minutes on the
available hardware. IVFFlat builds in seconds by clustering vectors into 100 Voronoi
cells, at the cost of requiring `probes` > 1 for high recall. The codebase uses pgvector's
default `probes = 1`, which is a known recall-quality improvement opportunity.

**REST API access pattern.** All database access goes through the REST API on
aio-01:5000. Workers, scrapers, and workflow agents never hold database connections
directly.

```
Worker/Agent --> POST http://aio-01:5000/... --> REST API --> pg.Pool --> PostgreSQL
```

*Why this constraint:* Connection pooling. PostgreSQL has a hard limit on concurrent
connections (default 100). With 8+ fleet workers, 88+ scrapers, and multiple workflow
agents running concurrently, direct connections would exhaust the pool. The REST API
centralizes connection management through a single `pg.Pool` (max 10 connections, 30s
idle timeout), serializing access and preventing connection storms.

*Tradeoff:* Every database operation incurs HTTP overhead (serialization, network round
trip, deserialization). For bulk operations, this adds 5-15ms per request. The system
accepts this latency in exchange for connection safety and operational simplicity.

**Migration strategy.** The `db/migrations/` directory contains 20+ numbered SQL migration
files (e.g., `031_hybrid_search_indexes.sql`). Migrations are applied manually -- there is
no automated migration runner. A full schema dump exists at
`ansible/files/sql/full-schema.sql` for bootstrapping new deployments.

**Why PostgreSQL over specialized databases.** We evaluated dedicated vector databases
(Pinecone, Weaviate, Milvus) and found that pgvector eliminates an entire infrastructure
component. With PostgreSQL, vector similarity, full-text search, relational joins, and
ACID transactions all happen in the same query plan. A query like "find chunks similar to
X that were ingested in the last 7 days from Python documentation" is a single SQL
statement, not a multi-database join.

*Limitation:* pgvector's HNSW implementation is single-threaded during index builds and
does not support GPU acceleration. For datasets beyond ~10M vectors, a dedicated vector
database would likely outperform it.


### 11.2 Redis

Redis serves three roles: pipeline queue management, idempotency enforcement, and worker
coordination. It runs on aio-01 (port 6379), no authentication, with Sentinel support
across three nodes for high availability.

**Queue implementation.** All queue operations use Lua scripts executed atomically via
`EVALSHA`. This is critical: without atomic claim-and-acknowledge, two workers could claim
the same item from a sorted set.

**Redis key structure per pipeline stage:**

```
pipeline:{stage}:pending          ZSET   Priority-scored work items
pipeline:{stage}:processing       HASH   Claimed items (item_id -> claimed_at)
pipeline:{stage}:completed_set    SET    Completed item IDs (7-day TTL)
pipeline:{stage}:completed_count  STRING Atomic counter
pipeline:{stage}:failed_count     STRING Terminal failure counter
pipeline:{stage}:dlq              LIST   Dead-letter queue
pipeline:item:{id}                HASH   Full item payload
pipeline:idempotency:{key}        STRING 24-hour TTL deduplication guard
pipeline:next_id                  STRING Sequential ID counter
```

**Seven Lua scripts** handle all mutations:

| Script | Operation | Atomicity Guarantee |
|--------|-----------|---------------------|
| `add` | ZADD to pending, SET idempotency key | Item added exactly once |
| `fetch` | ZPOPMIN from pending, HSET to processing | Item claimed by exactly one worker |
| `complete` | HDEL from processing, SADD to completed, ZADD to next stage | Item transitions atomically |
| `fail` | HDEL from processing, conditional ZADD (retry) or LPUSH (DLQ) | No item lost between states |
| `reclaim` | Scan processing for stale items, ZADD back to pending | Stuck items recovered without duplicates |
| `heartbeat` | Update timestamp in processing hash | Ownership verified before update |
| `purge` | Remove old completed/failed items | Bounded memory usage |

**Why Redis over PostgreSQL for queues.** The original implementation used PostgreSQL with
`SELECT ... FOR UPDATE SKIP LOCKED`. This worked but had two problems: lock contention
under high concurrency (8+ workers polling simultaneously), and vacuum pressure from
frequent row updates (claim/complete/delete cycles generate dead tuples). Redis sorted
sets provide O(log n) insertion and O(1) removal with no locking overhead and no vacuum
maintenance.

*Alternative considered:* RabbitMQ or Apache Kafka. Both would work but add operational
complexity (another service to deploy, monitor, and upgrade) for a workload that fits
comfortably in Redis's sorted set model. The priority scoring formula
(`(10 - priority) * 1e13 + timestamp_ms`) elegantly encodes two dimensions into a single
ZSET score, which neither RabbitMQ nor Kafka supports natively.

**Operational constants:**

| Constant | Value | Rationale |
|----------|-------|-----------|
| Idempotency TTL | 24 hours | Long enough to cover retry storms; short enough to allow reprocessing after a day |
| Completed item TTL | 7 days | Audit trail for debugging; auto-cleaned to bound memory |
| Stuck threshold | 600 seconds | 10 minutes -- long enough for slow embeddings, short enough to detect genuine failures |
| Max retries | 3 | Empirically, transient failures resolve within 3 attempts; persistent failures go to DLQ |


### 11.3 OrientDB

OrientDB serves as the knowledge graph for infrastructure relationships and document
similarity traversals. It runs in a Docker container on aio-01 (ports 2424 binary, 2480
HTTP REST), database name `orchestrator`.

**Node types and scale:**

| Vertex Class | Count | Purpose |
|-------------|-------|---------|
| Document | 315K+ | Scraped document chunks |
| Memory | Varies | Memory entries from auto-storage |
| Infrastructure | 9 | Fleet nodes |
| Project | Varies | Project entities |

**Edge types:**

| Edge | Direction | Properties | Purpose |
|------|-----------|------------|---------|
| RelatedTo | Document <-> Document | similarity (float) | Content similarity |
| BelongsTo | Memory -> Project | -- | Project membership |
| NextChunk | Memory -> Memory | -- | Sequential chunk ordering |
| EXECUTED_ON | Workflow -> Node | -- | Execution tracking |
| USED_MODEL | Workflow -> Model | -- | Model usage tracking |
| DEPENDS_ON | Workflow -> Document | -- | Document dependencies |
| CONNECTED_TO | Node -> Node | -- | Network topology |
| CitedBy | Document -> Document | -- | Citation relationships |

**Access pattern.** All OrientDB access goes through the REST API at
`POST http://aio-01:5000/graph/query` with OrientDB SQL in the request body. No
application code imports an OrientDB driver library -- the REST API is the only interface.

```sql
-- Multi-hop traversal: find documents related within 3 hops
SELECT FROM (
    TRAVERSE out('RelatedTo') FROM #12:0 MAXDEPTH 3
) WHERE @class = 'Document'

-- Shortest path between two infrastructure nodes
SELECT shortestPath(#9:0, #9:5, 'BOTH', 'CONNECTED_TO')
```

**Why OrientDB over Neo4j.** Neo4j was the original choice (and stale Neo4j client code
remains in the codebase). It was replaced due to persistent data loss issues in Docker
deployments -- OrientDB's built-in REST API eliminated the need for a language-specific
driver, and its multi-model capability (document + graph + key/value in one engine)
simplified the deployment.

**Why not just PostgreSQL for graph queries.** Consider finding all documents within 3
hops of a given document. In SQL:

```sql
-- Hop 1
SELECT d2.* FROM relationships r1
  JOIN documents d2 ON r1.target_id = d2.id
  WHERE r1.source_id = 'doc-A'
-- Hop 2: join again
-- Hop 3: join again
-- Each hop multiplies the result set
```

With 315K+ documents and potentially millions of relationships, each self-join scans a
growing intermediate result set. Benchmarks showed OrientDB completing 3-hop traversals
in under 10ms while the equivalent PostgreSQL query took 12.5 seconds.

**Limitation.** OrientDB is deployed via Ansible but is not included in the main
`site.yml` playbook -- it must be deployed manually. There is no automated backup for
OrientDB data (only PostgreSQL has daily backups). Since OrientDB can be rebuilt from
PostgreSQL, this is an accepted risk, but it means a container crash requires a manual
rebuild step.

---

## 12. Genetic Algorithm Evolution

The genetic algorithm (GA) system evolves configuration parameters for fleet orchestration,
model routing, RAG retrieval, team composition, and more. It operates on the principle
that good configurations are discoverable through evolutionary search over real execution
history.

**Core engine:** `learning/ga_engine.py`

**FleetChromosome.** The fundamental unit of evolution is a dataclass with 6 genes:

```python
@dataclass
class FleetChromosome:
    workers: int          # 1-6: parallel worker count
    arbiter: str          # Consensus model (from validated pool)
    timeout: int          # 30-600 seconds
    consensus_type: str   # majority | unanimous | weighted | ranked
    retry_limit: int      # 1-5 retries before failure
    diversity_weight: float  # 0.0-1.0: weight given to model diversity
```

**Genetic operators:**

```
Parent A:  [4, opus, 120, majority, 3, 0.7]
Parent B:  [6, sonnet, 300, weighted, 1, 0.3]
                  |           |
           Two-Point Crossover (positions 2, 4)
                  |           |
                  v           v
Child A:   [4, opus, 300, weighted, 3, 0.7]
Child B:   [6, sonnet, 120, majority, 1, 0.3]
                  |
           Mutation (rate 0.1 per gene)
                  |
                  v
Child A':  [4, opus, 300, ranked, 3, 0.7]
                        ^^^^^^^
                     (mutated gene)
```

*Selection:* Tournament selection with tournament size 3, drawn from the elite (top 50%
of the population by fitness).

*Validation:* Both crossover and mutation retry up to 10 times if offspring fail
validation constraints (e.g., worker count out of range). If all retries fail, parents
are returned unchanged -- the system never produces invalid chromosomes.

**Default hyperparameters:**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Population size | 20 | Large enough for diversity; small enough for convergence in 50 generations |
| Mutation rate | 0.1 | Per-gene probability; low enough to preserve good solutions |
| Generations | 50 | Empirically sufficient for convergence on 6-gene chromosomes |
| Elitism | Top 50% | Aggressive preservation of good solutions |
| Tournament size | 3 | Moderate selection pressure |

**Fitness function (weighted multi-objective):**

```
Fitness = success_rate * 0.4
        + avg_quality  * 0.3
        + efficiency   * 0.2
        + diversity    * 0.1
```

Fitness components are computed from real execution history queried via the REST API, not
from synthetic benchmarks. This grounds evolution in actual system performance.

**Seven specialized optimizers** extend the core engine for different parameter spaces:

| Optimizer | Chromosome | Genes | Pop | Gens | Key Fitness Components |
|-----------|-----------|-------|-----|------|----------------------|
| Model Routing | Task-to-model mapping | 7 task types | 30 | 1000 | quality (0.6), cost (0.2), latency (0.2) |
| RAG Retrieval | Chunk/embed/search config | 10 params | 40 | 1000 | relevance (0.30), coverage (0.20), efficiency (0.20), diversity (0.15), latency (0.15) |
| Team Composition | Model team selection | 5 models/team | 20 | 1000 | quality (0.5), diversity (0.3), cost (0.2) |
| Training Data | Category mix recipe | 8+ params | 30 | 1000 | coverage (0.3), balance (0.2), quality (0.2), volume (0.15), richness (0.15) |
| Adversarial Verification | Mutation strategies | Variable | 20 | 15 | evasion_score = (1 - detection_rate) * validity |
| Prompt Evolution | Review check sets | Variable | 20 | 10 | F1 = 2 * precision * recall / (precision + recall) |
| Workflow Config | Orchestration params | 6 params | 20 | 50 | cost (0.3), quality (0.4), duration (0.2), success (0.1) |

**Operator variations across optimizers:**

| Operator | Core Engine | Model Routing | RAG Retrieval | Team Selection |
|----------|------------|---------------|---------------|----------------|
| Crossover | Two-point | Single-point | Uniform (per-gene) | Set-based (union + sample) |
| Mutation | Single-gene | Single-gene | Adaptive (1-3 genes) | Single-gene swap |
| Elitism | 50% | 10% | 20% | 20% |
| Selection | Tournament (k=3) | Tournament (k=5) | Tournament (k=4) | Tournament (k=3) |

**Adaptive mutation.** The RAG retrieval optimizer increases its mutation rate from 0.25
up to 0.60 when the population stagnates (no fitness improvement for 5+ generations).
This is a well-known technique for escaping local optima: when the population converges
prematurely, higher mutation rates introduce fresh genetic material.

**Statistical rigor.** The adversarial verification and prompt evolution optimizers run
multiple seeds (5 and 10 respectively) and use control groups (50-100 random
configurations) for comparison. The prompt evolution optimizer additionally computes
bootstrap confidence intervals and z-test significance values to distinguish genuine
improvements from noise.

**GA-to-Thompson-Sampling bridge.** After evolution completes, the best chromosome is
converted into a named Thompson Sampling strategy:

```python
def integrate_with_thompson_sampling(self):
    best = self.population[0]  # Sorted by fitness
    strategy_name = f"ga_gen{self.generation}_{best.arbiter}"
    alpha_increment = int(best.fitness * 10)

    # POST to /ga/strategies
    # Creates or updates Beta(alpha, beta) for real-time exploitation
```

This is the critical integration point: GA discovers globally good configurations through
evolution; Thompson Sampling exploits them in real-time through Bayesian reward
estimation. The GA runs periodically (offline), while Thompson Sampling runs on every
model selection decision (online).

**Convergence tracking.** All GA optimizers report generation-level statistics (best
fitness, average fitness, diversity metrics) to `POST /ga/convergence` and
`POST /ga/best-solutions`, enabling centralized monitoring of evolutionary progress
across all 7 optimizer instances.

---

## 13. Thompson Sampling

Thompson Sampling is the real-time decision engine that selects models for each task. It
operates as a multi-armed bandit, maintaining a Bayesian posterior distribution over each
model's quality and sampling from it to balance exploration and exploitation.

**Core algorithm.** Each model is a "bandit arm" with a Beta distribution parameterized
by (alpha, beta), where alpha counts successes and beta counts failures:

```
For each candidate model m:
    sample_m ~ Beta(alpha_m, beta_m)
Select model with highest sample_m
```

The Beta distribution naturally encodes uncertainty: a model with (alpha=2, beta=2)
(uncertain) will occasionally sample higher than a model with (alpha=100, beta=10)
(well-known), ensuring that under-explored models get tried.

**Three implementations** serve different layers of the system:

| Implementation | File | Prior | Success Threshold | State | Special Feature |
|---------------|------|-------|-------------------|-------|-----------------|
| Primary | `thompson-sampling.js` | Beta(1,1) | quality >= 0.7 | PostgreSQL `learning.strategy_performance` | Cryptographic randomness (Johnk's algorithm) |
| Enhanced | `model-selector.js` | Beta(1,1) | quality >= 0.6 | SQLite `learning/db/learning.db` | Sliding window (100), KS-test drift detection |
| Multi-Objective | `dcab-router.js` | Beta(1,1) per dimension | Continuous | PostgreSQL `learning.model_performance` | 4 independent Beta distributions per model |

**Why three implementations.** Each targets a different operational concern:

1. *Primary:* Simple, robust model selection for general tasks. Uses PostgreSQL for
   persistence across sessions. Cryptographic randomness (via `crypto.randomBytes`)
   ensures sampling quality is not compromised by weak PRNGs.

2. *Enhanced:* Adapts to distribution shift. A Kolmogorov-Smirnov test compares recent
   performance (last 7 days) against the baseline (previous 30 days). If KS > 0.15,
   posteriors reset to uniform Beta(1,1), forcing re-exploration. The sliding window
   (last 100 pulls only) prevents ancient history from dominating the posterior.

3. *Multi-Objective:* Tracks four quality dimensions independently (success, quality,
   speed, cost), each with its own Beta posterior. Scalarization weights
   (0.5/0.3/0.2/0.0) combine them, with a +0.2 Pareto frontier bonus for models that
   are not dominated on all dimensions.

**Explore/exploit tradeoff.** The enhanced selector adds an explicit exploration bonus
for under-sampled models (fewer than 5 pulls):

```
bonus = EXPLORATION_BONUS * (1 - pulls / 5)    // EXPLORATION_BONUS = 0.1
```

This ensures new models get at least 5 trials before the system relies solely on their
posterior for selection.

**Integration with GA.** When the GA evolves a new best chromosome, it creates a named
strategy (e.g., `ga_gen50_opus`) and initializes its Beta distribution in the shared
`learning.strategy_performance` table. The Thompson Sampling selector treats GA-evolved
strategies identically to manually defined ones -- they compete on the same footing,
and their posteriors update based on real execution outcomes.

Conversely, the GA model routing optimizer loads Thompson Sampling priors as quality
estimates for its fitness function, using confidence-weighted blending:

```
quality = measured_quality * (1 - w) + prior_avg_reward * w
where w = prior_confidence * 0.4
```

This creates a virtuous cycle: GA discovers promising configurations, Thompson Sampling
exploits them and accumulates evidence, and the next GA generation uses that evidence to
make better fitness evaluations.

---

## 14. Model Routing

Model routing determines which LLM handles each task. The system uses a quality-first
approach: since all 200+ models are available on free tiers, cost is not a factor in
routing decisions.

**Quality-first scoring formula:**

```
quality_weight = capability * confidence * history * calibration
```

| Factor | Source | Range | Purpose |
|--------|--------|-------|---------|
| capability | Static task-model matrix | 0.0-1.0 | Known model strengths per task type |
| confidence | Thompson Sampling posterior | 0.0-1.0 | Bayesian quality estimate |
| history | Execution history (EWMA) | 0.0-1.0 | Exponentially-weighted recent performance |
| calibration | Confidence calibration module | 0.25-1.0 | Penalty for overconfident models |

**Task classification.** Tasks are classified into one of 7 canonical categories using
keyword-based heuristics:

```
code_generation   - "write", "implement", "create function"
code_review       - "review", "check", "audit"
research          - "research", "find", "investigate"
math_reasoning    - "calculate", "prove", "solve"
general_qa        - "explain", "describe", "what is"
creative_writing  - "write story", "draft", "compose"
security_analysis - "vulnerability", "CVE", "exploit"
```

**Contextual bandit (advanced).** The `integrated_router.py` implements a LinUCB
contextual bandit that goes beyond the simple task-type heuristic. It constructs a
128-dimensional context vector fusing 5 context sources (task, session, user, temporal,
resource) and computes an Upper Confidence Bound score:

```
UCB = theta^T * context + alpha * sqrt(context^T * A_inv * context)
```

The exploration parameter `alpha = 1.0` balances expected reward (exploitation) against
uncertainty (exploration). The `A_inv` matrix updates efficiently via the Sherman-Morrison
formula, avoiding full matrix inversion on every update.

**Fallback chains.** When the primary model is unavailable (circuit breaker open, rate
limited, API error), the router follows model-specific fallback chains:

```
opus   -> sonnet -> haiku
fable  -> opus   -> sonnet
gemini -> fable  -> sonnet
gpt-4o -> opus   -> sonnet
```

For provider-level failures, the chain crosses provider boundaries:

```
Groq -> Gemini -> Cerebras -> DeepInfra -> Together
```

**Consensus voting.** When multiple models produce answers for the same task, the router
groups responses by content similarity and sums quality weights per group:

| Consensus Level | Threshold | Meaning |
|----------------|-----------|---------|
| Strong | >= 0.80 | High confidence in the majority answer |
| Moderate | >= 0.60 | Reasonable confidence |
| Weak | >= 0.40 | Low confidence; consider additional models |
| No consensus | < 0.40 | Models fundamentally disagree; escalate |

**Rate limiting integration.** Before scoring, the router filters out models whose
providers have exceeded their rate limits (e.g., Groq at 30 RPM, OpenRouter at 10 RPM).
This prevents the router from selecting a model that will immediately fail.

---

## 15. Git Worktrees

Git worktrees provide filesystem-level isolation for concurrent AI agents working on the
same repository. Each agent operates on an independent copy of the working tree, with its
own branch, preventing merge conflicts from concurrent modifications.

**Pattern:**

```
main (shared)
  |
  +-- .claude/worktrees/issue-42/    branch: fix/issue-42
  |     (Agent A modifying files)
  |
  +-- .claude/worktrees/issue-57/    branch: fix/issue-57
        (Agent B modifying files)
```

Each worktree gets its own branch, its own working directory, and its own staging area.
Agents can modify, commit, and test without interfering with each other or with the main
branch.

**Two coexisting approaches.**

*Built-in isolation* (`isolation: 'worktree'` in workflow definitions): The Claude Code
framework handles worktree creation and cleanup automatically. Used in `code-solve.js`
and `pr-verify.js` workflows.

*Manual management* (in `code-solve-auto.js`): Creates worktrees at
`.claude/worktrees/issue-{N}` with explicit squash-merge-to-main flow and cleanup in
a `finally` block. This approach exists because the built-in mechanism had issues with
merge-back in parallel execution scenarios -- commits from worktrees were not reliably
merged back to main.

**GA-evolved isolation.** The self-learning resolver treats worktree isolation as a
genetic algorithm gene (`fix_isolation: cycle [shared -> worktree -> shared]`), allowing
the system to discover whether isolation helps or hurts for different issue types. Cold
start defaults to `shared` (no isolation).

**Worktree-aware exclusions.** Three code analysis tools explicitly exclude
`.claude/worktrees/` directories from their scans to avoid processing duplicated code
in active worktrees.

**Why worktrees over branches alone.** Git branches share a single working tree. If
Agent A checks out branch `fix/42` and starts modifying `server.py`, Agent B cannot
check out branch `fix/57` to modify the same file without overwriting Agent A's
uncommitted changes. Worktrees solve this at the filesystem level: each agent gets its
own copy of every file.

---

## 16. Anti-Feedback-Loop Safeguards

When LLMs evaluate other LLMs' outputs, and the evaluation results feed back into model
selection, the system risks collapsing into a self-reinforcing loop where a dominant model
evaluates its own work favorably, gets selected more often, evaluates itself more, and
converges on a single perspective. This section describes the detection and prevention
mechanisms.

**The risk model.** Four distinct failure modes, each caught by a dedicated detection
layer:

```
+------------------------------------------------------------------+
|                    FEEDBACK LOOP RISKS                            |
+------------------------------------------------------------------+
|                                                                  |
|  Layer 1: MODEL DOMINANCE                                        |
|  Risk: One model handles >70% of tasks                           |
|  Effect: Loss of diversity, single point of failure              |
|  Detection: Count model usage in monitoring.execution_summary    |
|  Severity: (percentage - 0.70) / (1.0 - 0.70), clamped to 1.0   |
|                                                                  |
|  Layer 2: EVALUATOR-GENERATOR COUPLING                           |
|  Risk: Model evaluates >40% of its own outputs                   |
|  Effect: Self-confirming bias (model rates itself highly)         |
|  Detection: Join worker_results with arbiter_decisions            |
|             where worker_model == arbiter_model                   |
|                                                                  |
|  Layer 3: REWARD HACKING                                         |
|  Risk: Quality metrics rise while diversity metrics fall          |
|  Effect: System optimizes for its own metrics, not real quality   |
|  Detection: Quality trend > 0.02 AND diversity trend < -0.1      |
|             AND recent quality > 0.85                             |
|             AND recent > baseline * 1.10                          |
|                                                                  |
|  Layer 4: CONCEPT COLLAPSE                                       |
|  Risk: Output embeddings converge (>0.90 cosine similarity)      |
|  Effect: All models producing near-identical outputs              |
|  Detection: NxN similarity matrix on output embeddings            |
|             Mean > 0.85 OR >20% pairs exceed 0.90                 |
+------------------------------------------------------------------+
```

**Threshold profiles:**

| Profile | Dominance | Coupling | Reward Hack | Collapse |
|---------|-----------|----------|-------------|----------|
| Default | 0.70 | 0.40 | 0.85 | 0.90 |
| Conservative | 0.80 | 0.50 | 0.90 | 0.95 |
| Aggressive | 0.60 | 0.30 | 0.80 | 0.85 |

**Severity classification and system response:**

| Severity | Score Range | Exit Code | System Action |
|----------|-------------|-----------|---------------|
| Critical | > 0.8 | 2 | Middleware can halt workflows |
| High | 0.6 - 0.8 | 1 | Warning logged, alert generated |
| Medium | 0.4 - 0.6 | 0 | Informational, no action |
| Low | <= 0.4 | 0 | Informational, no action |

**Workflow middleware.** The JavaScript adapter provides pre-flight and post-flight
hooks that integrate directly into the workflow execution pipeline:

```javascript
const { beforeWorkflow, afterWorkflow } = require('./shared/feedback-loop-adapter.cjs');

// Pre-flight: halt if critical risks detected (severity > 0.8)
await beforeWorkflow({ criticalOnly: true });

// ... workflow executes ...

// Post-flight: check with 1-day window, warn but don't halt
await afterWorkflow({ workflowId: execId, warnOnly: true });
```

`beforeWorkflow` acts as a circuit breaker: if the feedback loop optimizer reports
critical-severity risks, the workflow does not start. This prevents a degraded system
from producing more self-reinforcing outputs.

**Automated monitoring.** A cron job runs the full 4-layer analysis every 6 hours. Reports
are written to `~/.claude/reports/feedback-loops/latest.json` with 30-day retention.

**External validation as the ultimate safeguard.** No automated system can fully prevent
feedback loops -- an LLM evaluating whether other LLMs are in a feedback loop is itself
part of the loop. The system design acknowledges this explicitly: the human operator is
the final arbiter. All detection layers produce human-readable reports, and critical risks
generate alerts that require human acknowledgment.

**Design philosophy.** The system is designed to detect and warn, not to auto-correct.
Auto-correction (e.g., automatically resetting posteriors when dominance is detected)
risks oscillation: reset -> re-explore -> re-converge -> reset. Instead, the system
surfaces evidence and lets the operator decide whether the detected pattern is genuinely
harmful (e.g., one model dominates because it is genuinely better) or a true feedback
loop (e.g., one model dominates because it evaluates itself favorably).

---

## 17. Monitoring and Observability

The monitoring stack tracks fleet health, model performance, API availability, cost, and
drift across the distributed system.

**Prometheus exporters (3 instances):**

| Exporter | Port | Key Metrics |
|----------|------|-------------|
| Transparency exporter | 9090 | Bug tracking, fix rates, deployment status |
| Consensus exporter | 9101 | Consensus decisions, per-model cost, disagreement histograms, drift alerts |
| Fleet exporter | 9091 | 30+ metrics: per-node CPU/RAM/load, workflow duration/status, model error rates, estimated cost |

**Fleet health monitoring.** SSH-based health checks (`ssh -o ConnectTimeout=5 claude@{worker} echo OK`) run every 60 seconds against all fleet nodes. A worker is marked
unhealthy after 3 consecutive failures and enters a 300-second recovery wait before
re-probing. Auto-recovery attempts `systemctl restart sshd` on failed nodes.

**API health monitoring.** Each LLM provider (Anthropic, OpenAI, Google, OpenRouter) is
checked every 5 minutes with a 10-second timeout. A 20-attempt sliding window determines
status:

| Status | Success Rate | Action |
|--------|-------------|--------|
| Healthy | > 75% | Normal routing |
| Degraded | 50-75% | Circuit breaker notified, reduced traffic |
| Disabled | < 50% | Auto-disabled, no routing |

Health status persists in `monitoring.api_health_status` for trend analysis and is
integrated with the circuit breaker to automatically remove degraded providers from the
routing pool.

**Model performance drift detection.** A 30-day rolling performance baseline detects
quality degradation exceeding 10% (with a minimum of 20 samples to reduce false
positives). The `monitoring.model_drift` materialized view and `monitoring.drift_alerts`
table track detected drift events. Alerts support acknowledgment workflow via
`--acknowledge=ID` for operational management.

**Cost tracking.** A dedicated cost tracker service (port 9094) enforces budget limits:
$5.00 maximum per workflow, $25.00 maximum per session, with warnings at 80% of each
threshold. Since all models use free tiers, cost tracking serves primarily as an anomaly
detector -- unexpected costs indicate misconfiguration or accidental use of paid models.

**Grafana dashboards.** Eleven dashboards cover fleet overview, consensus decisions,
issue tracking, ML model performance, database metrics, data pipeline status, document
ingestion, GA training progress, and system metrics. Grafana runs on aio-01:3000.

---

## 18. Security Considerations

**API key management.** Keys are encrypted at rest using AES-256-GCM via
`api-key-manager.cjs`. The encryption key is loaded from the `API_KEY_ENCRYPTION_KEY`
environment variable (64-character hex string / 32 bytes). Key validation in the document
ingestion API uses constant-time bcrypt comparison with fake bcrypt work on invalid keys
to prevent timing side-channel attacks. Keys are stored as bcrypt hashes in `auth.api_keys`
with expiration support.

*Limitation:* The fallback key loading path reads from `~/.claude/credentials.json` as
plaintext on disk. There is no automated key rotation.

**Red Hat code restrictions.** The `model-compliance-enforcer.js` module classifies code
by filesystem path. Any file under `/home/sfloess/Development/redhat` is treated as
proprietary. For proprietary code, only Anthropic models (Claude Fable, Opus, Sonnet,
Haiku) are permitted. All other providers (OpenAI, Google, DeepSeek, Cerebras, Groq,
OpenRouter, Cloudflare Workers AI) are blocked. The `model-compliance.js` module reads
path restrictions from `~/.claude/fleet.json` and exposes `filterAllowedModels()` and
`getCompliantWorkers()` for use in routing decisions.

**Input validation.** Two validation modules cover different threat vectors:

- `input-validator.js`: HTML stripping (script/style/event handler removal), path
  traversal blocking (rejecting `..` and null bytes), directory allowlisting, forbidden
  file pattern detection (`.env`, `secrets.json`, `.ssh/`), model name allowlisting,
  1MB body size limit, and prototype pollution detection (`__proto__`, `constructor`).

- `input-validation.cjs`: SQL injection keyword detection, command injection pattern
  detection (`$(`, backtick, `eval(`), and domain-specific validators (confidence
  0.0-1.0, worker count 1-16, duration max 1 hour, cost max $100).

**Rate limiting.** Per-provider sliding window rate limiting is implemented in
`rate-limit-manager.cjs` using PostgreSQL-backed counters. Provider-specific limits:
Groq 30 RPM, Perplexity 5 RPH, OpenRouter 10 RPM, default 60 RPM. A safety buffer
throttles at ~93% of the limit (e.g., 28/30 for Groq) to prevent burst-induced
rejections.

---

## 19. Scaling

The architecture is designed for incremental expansion along five dimensions. Each
dimension can scale independently without requiring changes to the others.

**Adding scrapers.** A new knowledge domain requires only a new `BaseScraper` subclass
with a `SOURCES` dict and a `scrape()` method. The 4-stage pipeline handles everything
downstream: the new scraper enqueues to the store queue, and the existing store/chunk/
embed/graph workers process the content identically to all other domains. Deploying the
new scraper to fleet workers uses the existing Ansible role.

**Adding models.** New free-tier models on any supported provider (OpenRouter, Anthropic,
Google, Groq, Cerebras, DeepSeek) are immediately available for routing. The Thompson
Sampling system assigns them a uniform prior Beta(1,1) and begins exploration
automatically. The GA model routing optimizer will incorporate them in its next evolution
cycle. No configuration change is needed -- the system discovers and evaluates new models
through the existing bandit mechanism.

**Adding fleet nodes.** A new node requires SSH access for the `claude` user and
registration with the circuit breaker. The fleet health monitor begins probing it
automatically. Workers are dispatched via the existing SSH-based execution model. Memory
constraints determine which workloads the node can handle: 1GB nodes handle I/O-bound
orchestration; 8GB+ nodes can run embedding or cross-encoder workloads.

**Adding GA optimizers.** Each optimizer is self-contained: define a chromosome (the
parameter space), a fitness function (how to evaluate), and operators (how to combine and
mutate). The existing GA engine provides tournament selection, crossover scaffolding, and
convergence tracking. A new optimizer registers its convergence data via the same
`POST /ga/convergence` endpoint, appearing automatically in monitoring dashboards.

**Adding review layers.** The multi-panel review architecture (review panel, meta-review
panel, arbiters) supports additional layers without structural changes. A new review
layer requires only defining a model panel with zero overlap against existing panels and
designating an arbiter. The consensus voting mechanism handles any number of input
panels.

**Scaling limits.** PostgreSQL is the primary bottleneck: the single-instance deployment
on aio-01 (10-connection pool) bounds maximum concurrent database operations. Redis
(single-instance, no clustering) bounds maximum queue throughput. OrientDB
(single-container, no replication) bounds maximum graph query concurrency. All three are
adequate for the current 9-node fleet but would require sharding or replication for
significantly larger deployments.

---

## 20. Failure Recovery

The system handles failures at four levels: individual model calls, provider outages,
worker node failures, and pipeline processing failures.

**Circuit breaker pattern.** Four independent circuit breaker implementations protect
different system boundaries:

| Implementation | Threshold | Reset Timeout | States | Scope |
|---------------|-----------|---------------|--------|-------|
| MCP fleet orchestrator | 5 failures | 30s | CLOSED, OPEN | Per-worker |
| Retry library | 5 failures | 60s | CLOSED, OPEN, HALF_OPEN | Per-endpoint |
| Recovery chain | 5 failures (min 2) | 300s | closed, open, half-open | Per-service |
| Orchestrator-level | 3 failures | 60s | Per-database | Infrastructure |

The retry library uses epoch-based atomic transitions for thread safety and supports a
HALF_OPEN state that allows a single probe request through. Non-retryable HTTP status
codes (400, 401, 403, 404, 405, 409, 422) bypass the circuit breaker entirely -- these
are client errors that will not resolve with retry.

**Provider fallback chains.** A 5-step recovery sequence handles model failures:

```
1. Primary model attempt
   |
   v (failure)
2. Model-specific fallback (opus -> sonnet -> haiku)
   |
   v (failure)
3. Retry with exponential backoff (3 retries, 1s base, 30s max)
   |
   v (failure)
4. Cross-provider fallback (Groq -> Gemini -> Cerebras -> DeepInfra -> Together)
   |
   v (failure)
5. NEVER SKIP - fail the task explicitly, never produce empty output
```

The "never skip" principle is documented with empirical backing: skipping a failed step
and returning an empty result produces a 24% reward score (catastrophic), while failing
explicitly and retrying later produces a 68% reward score on average.

An ML-trained classifier (`shared/error_recovery.py`, 99.2% accuracy) predicts whether
an error is retryable and recommends the best fallback model based on the error type
(timeout, rate limit, connection error, server error).

**Redis queue retry and dead-letter routing.** All retry logic is implemented in Lua
scripts for atomicity:

```
On task failure:
  if retries < max_retries (3):
      ZADD back to pending with incremented retry count
      (preserves original priority)
  else:
      LPUSH to dlq:{stage}
      (terminal failure, requires manual intervention)
```

Dead-letter queue items are never automatically retried. They represent persistent
failures (malformed content, unsupported formats, upstream API changes) that require
human diagnosis.

**Idempotency.** Three levels of duplicate prevention:

1. *Queue-level:* Redis idempotency keys (24-hour TTL) prevent re-enqueuing completed
   items.
2. *Worker-level:* CTE queries with `NOT EXISTS` clauses skip items whose idempotency
   key has a completed entry.
3. *Optimistic concurrency:* A `work_version` column enables compare-and-swap updates,
   preventing two workers from completing the same item.

**Worker heartbeats and stuck-item reclaim.** Workers send heartbeats every 30 seconds
(PostgreSQL) or maintain a 5-minute TTL in Redis processing metadata. A stuck task
recovery daemon runs every 60 seconds, scanning for items whose heartbeat has expired.
Stuck items are requeued to the pending set with recovery metadata (original worker ID,
stuck duration, reclaim timestamp).

**Node lifecycle management.** Fleet nodes transition through states: unknown -> healthy
-> degraded (2 consecutive failures) -> failed (circuit breaker opens) -> recovering
(exponential backoff probing: 30s base, 2x multiplier, 2-minute max) -> healthy. Nodes
that return from failure undergo session resurrection -- their pending work is
redistributed and they re-enter the routing pool.

**Minimum viable fleet.** If active workers drop below 50% of the total fleet, an alert
fires. The system continues operating with reduced parallelism rather than halting.
PostgreSQL advisory locks prevent concurrent degradation checks from producing
conflicting assessments.

---

## 21. Future Evolution

The architecture is designed for incremental expansion. Each subsystem -- scrapers,
models, fleet nodes, GA optimizers, review layers, pipeline stages -- can grow
independently without structural changes to the others.

**Near-term expansion vectors:**

- *More scrapers, more domains.* The BaseScraper pattern makes adding a new knowledge
  domain a single-file change. The pipeline handles the rest.

- *More models, more providers.* As new free-tier models appear on OpenRouter and other
  providers, the Thompson Sampling system discovers and evaluates them automatically.

- *More fleet nodes.* SSH + circuit breaker registration is the only requirement.
  Heterogeneous hardware is supported -- nodes self-select for workloads based on their
  capabilities.

- *More GA dimensions.* Each new tunable parameter (chunk sizes, overlap ratios, re-ranking
  thresholds, consensus voting weights) can be added as a gene in an existing chromosome
  or as a new optimizer with its own chromosome definition.

- *More review layers.* The multi-panel review architecture supports any number of
  independent review stages, each with zero model overlap against the others.

**Structural constraints that would require architectural change:**

- *Multi-region deployment:* The current single-LAN architecture assumes low-latency
  access to PostgreSQL, Redis, and OrientDB. Cross-region deployment would require
  database replication, queue federation, and latency-aware routing.

- *Datasets beyond ~10M vectors:* pgvector's single-node architecture would need to be
  supplemented or replaced with a distributed vector database.

- *Real-time streaming workloads:* The current batch-oriented pipeline (scrape -> queue
  -> process) is not designed for sub-second latency. A streaming architecture
  (e.g., Kafka-based) would be needed for real-time knowledge ingestion.

The architecture does not attempt to solve these problems preemptively. Each represents a
scaling threshold that is far beyond current operational needs, and premature optimization
for them would add complexity without near-term benefit. When the time comes, each can be
addressed incrementally: add a Kafka stage alongside the existing Redis queues, add a
Milvus cluster alongside pgvector, add a cross-region proxy alongside the existing REST
API.
