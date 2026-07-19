# Design Philosophy

This document explains the engineering reasoning behind every major architectural decision in the FlossWare Distributed LLM Orchestration Framework. Each section states the problem, the decision, the alternatives we considered, the tradeoffs we accepted, and what would need to change for us to reconsider.

---

## Why Orchestration Instead of Training

**Problem:** Large language models are expensive to train, require specialized hardware, and deprecate rapidly as frontier labs release better checkpoints. A system that depends on training its own models is perpetually behind.

**Decision:** Treat models as pre-trained black boxes. All improvements come from better coordination: smarter routing, multi-model consensus, adversarial review, and iterative retry logic. The system never modifies model weights.

**Alternatives considered:**
- **Fine-tuning on commodity hardware.** We benchmarked CPU-based fine-tuning of 7B-parameter models. A single training run took 1-3 days and rendered the host machine unusable. GPU rental (RunPod, Lambda) costs $2/hour for an A100, making it viable per-run but operationally complex for continuous improvement. API-based fine-tuning (OpenAI, Google) costs $30-50 per run and limits you to the provider's model family.
- **Distillation pipelines.** Training a smaller model to mimic a larger one requires a stable teacher model, a curated dataset, and GPU time. The resulting student model is frozen at the teacher's capability level, so it falls behind when the teacher is updated.

**Tradeoffs accepted:** We cannot specialize models for domain-specific vocabulary or proprietary data formats. Prompt engineering and retrieval-augmented generation must compensate. We also cannot control inference internals (attention patterns, decoding strategies) beyond what the API exposes.

**Reconsider if:** Fine-tuning APIs become cheap enough (<$5/run) and support continuous incremental updates without full retraining, or if a task domain has no adequate pre-trained model coverage.

---

## Why API-Only

**Problem:** Hosting local models (Ollama, vLLM, llama.cpp) requires dedicated GPU hardware, model storage (7-70GB per model), version management, and operational burden. The hosted models lag behind frontier releases by weeks or months.

**Decision:** Access all models exclusively through provider APIs. The fleet currently reaches 200+ models across OpenRouter, Anthropic, Google, Groq, Cerebras, DeepSeek, and others.

**Alternatives considered:**
- **Local Ollama cluster.** We ran this configuration from project inception through June 2026. It worked for small models (7B-13B) but required NFS-mounted model storage, manual version updates, and consumed significant RAM on fleet workers. Quality was consistently below API-served frontier models.
- **Hybrid (local + API).** Adds routing complexity: the system must decide when a local model is "good enough" versus when to pay for an API call. Since all API models we use are on free tiers, the cost argument for local models disappeared.

**Tradeoffs accepted:** Complete dependency on external providers. If OpenRouter goes down and Google and Anthropic are also unavailable simultaneously, the system cannot function. We mitigate this with provider redundancy (78+ fallback providers) and circuit breakers, but a global API outage would be a hard stop.

**Reconsider if:** We need sub-10ms inference latency (local models eliminate network round-trips), we need to process data that cannot leave the network for compliance reasons, or API free tiers are eliminated.

---

## Why REST as the Integration Protocol

**Problem:** The orchestration layer must be callable from any language, any workflow script, any CI/CD pipeline, and any future component we have not yet imagined.

**Decision:** A Flask-based REST API is the single integration surface. Every capability -- model routing, consensus, fleet dispatch, workflow storage, learning queries -- is exposed as an HTTP endpoint.

**Alternatives considered:**
- **gRPC.** Higher throughput and type-safe contracts via Protocol Buffers. But requires code generation per client language, is harder to debug with curl, and adds a compilation step to the development workflow.
- **Message queues (RabbitMQ, Kafka).** Appropriate for fire-and-forget workloads but adds infrastructure (broker deployment, topic management) and complicates request-response patterns that our workflows rely on.
- **Direct library imports.** The tightest coupling and the highest performance, but locks every consumer to Python and the same process boundary.

**Tradeoffs accepted:** HTTP overhead (connection setup, JSON serialization) adds latency compared to gRPC or in-process calls. For our workloads (LLM inference taking 2-30 seconds per call), the 1-5ms of HTTP overhead is negligible.

**Reconsider if:** We need streaming intermediate results during inference (WebSockets or SSE would supplement REST, not replace it), or if a component requires sub-millisecond IPC.

---

## Why PostgreSQL

**Problem:** The system needs transactional storage (workflow state, cost tracking), vector similarity search (experience retrieval, semantic deduplication), and analytical queries (model performance aggregation, cost reports) -- ideally without operating three separate databases.

**Decision:** PostgreSQL with the pgvector extension. One database handles ACID transactions, 128/384-dimensional vector search, and complex SQL analytics.

**Alternatives considered:**
- **ChromaDB.** Purpose-built for vector search. In our benchmarks, pgvector was 2x faster for simple similarity queries (0.44ms vs 0.9ms) and 5x faster for filtered similarity queries (0.44ms vs 2.3ms) on 128-dimensional vectors. ChromaDB also requires a separate process and has no transactional guarantees.
- **Pinecone / Weaviate / Milvus.** Managed vector databases with excellent scaling characteristics, but they introduce an external service dependency and cannot handle our relational workloads (cost tracking, workflow state machines, strategy performance tables).
- **SQLite.** Embedded and zero-configuration, but single-writer and no native vector support. Unsuitable for concurrent fleet operations.

**Tradeoffs accepted:** pgvector is not optimized for billion-scale vector search. At our current scale (tens of thousands of experiences), this is irrelevant. PostgreSQL also requires operational maintenance (backups, vacuuming, connection pooling) that SQLite would not.

**Reconsider if:** Vector corpus exceeds 10 million rows and query latency becomes unacceptable, or if we need approximate nearest neighbor with sub-millisecond latency at scale (would add a dedicated vector index like FAISS as a cache layer, not replace PostgreSQL).

---

## Why Redis

**Problem:** Queue operations, rate limiting, and caching require sub-millisecond latency and atomic operations. PostgreSQL can technically serve as a queue (SELECT FOR UPDATE SKIP LOCKED), but polling-based queues waste connections and add latency.

**Decision:** Redis for all queuing (ingestion, embedding), rate limiting (per-provider token buckets), and short-lived caching (consensus results, health check state). Deployed with Sentinel for high availability across three nodes.

**Alternatives considered:**
- **PostgreSQL LISTEN/NOTIFY.** Native to our existing database but lacks the data structure primitives (sorted sets for priority queues, Lua scripting for atomic multi-step operations) that Redis provides.
- **RabbitMQ.** Full-featured message broker with acknowledgment semantics, dead-letter queues, and routing. Adds significant operational complexity (Erlang runtime, management UI, cluster configuration) for workloads that Redis handles with a single BRPOPLPUSH.
- **In-process queues.** Zero latency but no persistence across restarts and no cross-process visibility.

**Tradeoffs accepted:** Redis is an additional infrastructure component to operate. Data in Redis is ephemeral by design; if the Redis node restarts, queued items are lost (mitigated by Sentinel failover and AOF persistence). Redis is not suitable for durable storage -- that remains PostgreSQL's role.

**Reconsider if:** We need guaranteed exactly-once delivery semantics (would add RabbitMQ or Kafka for that specific workload), or if Redis memory usage exceeds available RAM on the Sentinel nodes.

---

## Why OrientDB

**Problem:** Infrastructure relationships (machine-runs-service, service-depends-on-database, network-connects-machines) are naturally graph-shaped. Querying these relationships in SQL requires self-joins that grow exponentially with traversal depth.

**Decision:** OrientDB as a multi-model (graph + document) database for knowledge graphs and infrastructure topology. Deployed on aio-01 via Docker.

**Alternatives considered:**
- **Neo4j.** The most popular graph database. We ran it initially but encountered persistent Docker volume corruption that caused data loss across container restarts. The community edition also lacks clustering.
- **ArangoDB.** Multi-model like OrientDB. We evaluated it and found comparable performance. OrientDB's SQL-like query language (SELECT EXPAND(out('runs')) FROM Machine WHERE name='aio-01') was more approachable for the team.
- **PostgreSQL recursive CTEs.** Technically capable of graph traversal but syntactically painful and slow beyond 3-4 hops.

**Tradeoffs accepted:** OrientDB is a less widely adopted database, which means a smaller ecosystem of tooling, fewer Stack Overflow answers, and a steeper learning curve for new contributors. Docker deployment adds a layer of operational complexity.

**Reconsider if:** OrientDB's maintenance burden becomes unsustainable, or if PostgreSQL's graph extensions (Apache AGE) mature enough to handle our traversal patterns without the performance cliff.

---

## Why Multi-Model Consensus

**Problem:** Every LLM has systematic blind spots. Claude excels at nuanced reasoning but can be overly cautious. GPT-4o is strong at structured output but sometimes hallucinates confidently. Gemini handles long contexts well but can miss subtle logical errors. No single model is reliably best across all task types.

**Decision:** Query 3-8 models per task, then synthesize responses through an arbiter model. The arbiter scores each response on relevance, correctness, and completeness, then produces a unified answer.

**Alternatives considered:**
- **Single best model.** Simpler and faster. But "best" varies by task type, and a single model's failure mode is undetectable without a reference point.
- **Ensemble averaging.** Works for numeric outputs but not for natural language. You cannot average two paragraphs of text.
- **Majority vote.** Appropriate for classification tasks but loses nuance for open-ended generation. A correct minority opinion would be silenced.

**Tradeoffs accepted:** Consensus is slower (3-8 parallel API calls + arbiter synthesis) and consumes more API quota. For latency-sensitive tasks, we offer a single-model fast path. Consensus also introduces arbiter bias: the arbiter's own blind spots can distort synthesis.

**Reconsider if:** A single model demonstrably dominates all others across all task types (historically, this has never been true for more than a few weeks after a new model release).

---

## Why Adversarial Review with Zero Model Overlap

**Problem:** If the same models that generate a finding also review it, they tend to confirm their own reasoning. This is self-referential validation -- the system optimizes for internal coherence rather than external correctness.

**Decision:** Maintain two non-overlapping model panels. The review panel (e.g., Opus, Sonnet, DeepSeek, Qwen) finds issues. The meta-review panel (e.g., Fable, Hermes-405B, Nemotron-Ultra, Qwen3-Next) adversarially challenges those findings. Zero model overlap between panels.

**Alternatives considered:**
- **Same-model self-review.** Fastest but nearly useless. Models rarely catch their own errors.
- **Partial overlap.** Allows some models to appear in both panels. Reduces the number of models needed but reintroduces confirmation bias proportional to the overlap.
- **Human-only review.** The gold standard for correctness but does not scale. Human review is reserved for final validation of high-stakes changes.

**Tradeoffs accepted:** Maintaining two strong panels requires access to at least 8 capable models. If model availability drops, we may not be able to staff both panels with sufficiently strong reviewers. Weak meta-reviewers (e.g., using Haiku to challenge Opus findings) provide false confidence.

**Reconsider if:** A formal verification system can replace adversarial review for specific task types (e.g., code correctness via proof assistants).

---

## Why Thompson Sampling

**Problem:** With 200+ models available, statically assigning models to task types is brittle. Model quality changes with provider updates, and new models enter the pool regularly. The system needs to balance exploiting known-good models with exploring potentially better alternatives.

**Decision:** Thompson Sampling, a Bayesian multi-armed bandit algorithm. Each model-task pair maintains a Beta distribution parameterized by successes and failures. At selection time, the system samples from each distribution and picks the model with the highest sample. This naturally balances exploration (new or uncertain models get tried) and exploitation (proven models get selected more often).

**Alternatives considered:**
- **Epsilon-greedy.** Pick the best model (1 - epsilon) percent of the time, random model epsilon percent. Simpler but wastes exploration budget on clearly bad models and has no notion of uncertainty.
- **UCB (Upper Confidence Bound).** Deterministic exploration based on confidence intervals. Tends to over-explore in high-dimensional spaces (200+ models x 15 task types = 3000 arms).
- **Static routing tables.** Manually curated model-to-task assignments. Requires constant human maintenance and cannot adapt to provider changes.

**Tradeoffs accepted:** Thompson Sampling requires maintaining per-arm state (alpha/beta parameters) and is more complex to implement than epsilon-greedy. Cold-start for new models means initial assignments are essentially random until enough data accumulates. We mitigate cold-start by initializing new models with prior parameters based on their model family (e.g., a new Claude variant inherits the Claude family's priors).

**Reconsider if:** The number of arms exceeds 10,000 (would need hierarchical or contextual bandits), or if task classification becomes unreliable (Thompson Sampling assumes correct arm identification).

---

## Why Genetic Algorithms

**Problem:** The system has high-dimensional configuration spaces that resist manual tuning: model team composition (which 6 of 200 models to assign to a workflow), RAG retrieval parameters (chunk size, overlap, top-k, reranking threshold), and training data curation (which examples to include in few-shot prompts).

**Decision:** Genetic algorithms (GA) using real execution data as the fitness function. Populations of configurations evolve through selection, crossover, and mutation. Fitness is measured by actual task outcomes (code review accuracy, research quality scores, consensus agreement rates).

**Alternatives considered:**
- **Grid search.** Exhaustive but combinatorially infeasible. Selecting 6 models from 200 has over 3 billion combinations.
- **Bayesian optimization (e.g., Optuna).** Efficient for continuous, low-dimensional spaces. Our configuration spaces are discrete and high-dimensional (model identities, categorical parameters), where GAs have an advantage.
- **Manual tuning.** Works for small parameter sets but does not scale to the number of knobs we have.

**Tradeoffs accepted:** GAs require many evaluations to converge (typically 50-200 generations). Each evaluation involves real workflow execution, which consumes API quota and wall-clock time. We run GA optimization as background jobs during off-peak hours to minimize impact.

**Reconsider if:** The fitness evaluation becomes too expensive (would switch to surrogate-model-based optimization), or if the configuration space becomes smooth and continuous (would switch to Bayesian optimization).

---

## Why Git Worktrees

**Problem:** Multiple agents working on the same repository simultaneously will create merge conflicts, overwrite each other's changes, and corrupt working tree state.

**Decision:** Each agent operates in its own git worktree -- a separate working directory backed by the same repository. Worktrees share the object store but have independent HEAD, index, and working tree. When an agent finishes, its changes are committed on a branch and merged through normal git workflow.

**Alternatives considered:**
- **Separate clones.** Full repository copies. Wastes disk space (duplicated object store) and requires explicit fetch/push to synchronize.
- **Locking (file-level or branch-level).** Serializes agent work, eliminating the parallelism that makes multi-agent execution valuable.
- **Container isolation.** Each agent runs in a Docker container with its own filesystem. Adds container orchestration overhead and makes git history harder to manage.

**Tradeoffs accepted:** Worktrees cannot check out the same branch simultaneously. Each agent must work on a unique branch, which requires a branch naming convention and cleanup discipline. Worktrees also share the git lock file, so simultaneous git operations (gc, repack) can block.

**Reconsider if:** Agent count exceeds 20 simultaneous workers (worktree overhead becomes noticeable), or if the repository is a monorepo large enough that worktree creation time matters.

---

## Why Provider Independence

**Problem:** Depending on a single LLM provider creates a single point of failure. Providers have outages, change pricing, deprecate models, and modify rate limits without notice.

**Decision:** The system abstracts all model access behind a provider-agnostic routing layer. Models are identified by capability profile, not by provider. If Anthropic is down, the router automatically falls back to equivalent models on OpenRouter, Google, or Groq. The system currently has fallback chains across 78+ providers.

**Alternatives considered:**
- **Single-provider commitment.** Simpler integration, potentially better pricing, dedicated support. But a single outage halts all operations.
- **Dual-provider (primary + backup).** Covers the most common failure mode but does not handle correlated outages or provider-specific model deprecation.

**Tradeoffs accepted:** Provider abstraction adds a routing layer that must be maintained as APIs evolve. Different providers have different token limits, context windows, and response formats, so the abstraction is necessarily leaky. Testing must cover all active providers, which multiplies the test matrix.

**Reconsider if:** A provider offers capabilities so unique that no fallback exists (e.g., a proprietary model with no equivalent elsewhere), in which case we would add it as a non-redundant tier with explicit user warnings about availability.
