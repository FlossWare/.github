# Orchestration Layer

The orchestration layer is a Flask-based REST API that serves as the single integration surface for all system capabilities. It receives task requests, classifies them, selects appropriate models, distributes work across the fleet, and returns synthesized results.

---

## Request Lifecycle

Every request follows the same pipeline, regardless of task type.

```
                         ORCHESTRATION REQUEST LIFECYCLE
 
  Client                    Orchestrator (aio-01:5000)                     Fleet
  ------                    --------------------------                     -----
    |                                 |                                      |
    |  POST /api/consensus            |                                      |
    |------------------------------->|                                      |
    |                                 |                                      |
    |                    1. Validate & authenticate                          |
    |                    2. Classify task type                               |
    |                       (15 categories)                                  |
    |                    3. Check consensus cache                            |
    |                       (hit? return cached)                             |
    |                    4. Select models                                    |
    |                       (Thompson Sampling)                              |
    |                    5. Check circuit breakers                           |
    |                       (skip tripped providers)                         |
    |                                 |                                      |
    |                                 |  SSH distribute to workers           |
    |                                 |------------------------------------->|
    |                                 |                                      |
    |                                 |     Worker 1: model A response       |
    |                                 |<-------------------------------------|
    |                                 |     Worker 2: model B response       |
    |                                 |<-------------------------------------|
    |                                 |     Worker 3: model C (timeout)      |
    |                                 |<------------- X (circuit break) -----|
    |                                 |                                      |
    |                    6. Collect responses                                |
    |                       (wait for quorum or timeout)                     |
    |                    7. Synthesize via arbiter                           |
    |                       (score, weight, merge)                           |
    |                    8. Cache consensus result                           |
    |                    9. Store execution metadata                         |
    |                       (PostgreSQL)                                     |
    |                   10. Update Thompson Sampling                         |
    |                       (success/failure per model)                      |
    |                                 |                                      |
    |  200 OK { consensus, scores }   |                                      |
    |<-------------------------------|                                      |
```

**Step 1 - Validate.** Check request schema, API key (if configured), and rate limits for the calling client.

**Step 2 - Classify.** A lightweight classifier assigns the task to one of 15 categories. Classification drives model selection, worker count, and timeout configuration.

**Step 3 - Cache check.** Hash the task description and parameters. If an identical request was processed within the cache TTL (default: 5 minutes), return the cached consensus immediately. Cache is stored in Redis.

**Step 4 - Model selection.** Thompson Sampling draws from the Beta distribution for each model-task pair. The top N models (typically 3-8, depending on task category) are selected. Models with tripped circuit breakers are excluded before sampling.

**Step 5 - Circuit breaker check.** Each provider and each individual model maintain independent circuit breaker state. If a provider has exceeded its error threshold, all models on that provider are skipped.

**Steps 6-7 - Collection and synthesis.** Workers execute in parallel via SSH. The orchestrator waits for a quorum (default: majority of dispatched workers) or a timeout (default: 60 seconds, configurable per task category). The arbiter model receives all collected responses and produces a synthesis.

**Steps 8-10 - Post-processing.** The consensus result is cached in Redis, execution metadata is written to PostgreSQL (`workflow.executions`, `monitoring.execution_summary`), and Thompson Sampling parameters are updated based on outcome.

---

## Blueprint Organization

The Flask API is organized into 22 modular blueprints, each handling a distinct functional area. Blueprints are registered at startup and share common middleware for authentication, rate limiting, and error handling.

| Blueprint | Prefix | Responsibility |
|-----------|--------|----------------|
| consensus | /api/consensus | Multi-model consensus queries |
| fleet | /api/fleet | Worker registration, health, dispatch |
| models | /api/models | Model registry, capability profiles |
| routing | /api/routing | Thompson Sampling state, routing overrides |
| workflows | /api/workflows | Workflow execution, phase tracking |
| learning | /api/learning | Experience storage, strategy performance |
| costs | /api/costs | Token usage, cost tracking |
| monitoring | /api/monitoring | Execution logs, alerts, diversity metrics |
| graph | /api/graph | OrientDB graph queries |
| embeddings | /api/embeddings | Vector embedding generation |
| scraping | /api/scraping | Web scraping pipeline management |
| keys | /api/keys | API key management (21 providers) |
| health | /api/health | Liveness, readiness probes |
| circuit | /api/circuit | Circuit breaker state management |
| cache | /api/cache | Consensus cache inspection, invalidation |
| feedback | /api/feedback | Feedback loop analysis |
| queue | /api/queue | Redis queue inspection |
| providers | /api/providers | Provider status, fallback chains |
| tasks | /api/tasks | Task classification configuration |
| teams | /api/teams | Model team composition (GA-optimized) |
| eval | /api/eval | Evaluation harness invocation |
| admin | /api/admin | Administrative operations |

Each blueprint is a self-contained Python module with its own routes, request validation, and business logic. Blueprints communicate through shared services (database adapters, Redis clients, model registry) rather than importing each other directly.

---

## Task Classification

The classifier assigns incoming tasks to one of 15 categories. Classification determines which models are eligible, how many workers to dispatch, what timeout to apply, and which arbiter to use.

| Category | Workers | Timeout | Example |
|----------|---------|---------|---------|
| code_generation | 4-6 | 90s | "Write a Python function that..." |
| code_review | 4 + 4 | 120s | "Review this diff for bugs" |
| research | 6-8 | 120s | "What are the tradeoffs of..." |
| math_reasoning | 4-6 | 60s | "Prove that..." |
| security_audit | 4 + 4 | 120s | "Analyze this code for vulnerabilities" |
| summarization | 3-4 | 30s | "Summarize this document" |
| classification | 3-4 | 15s | "Categorize this issue" |
| translation | 3-4 | 30s | "Translate to French" |
| data_extraction | 3-4 | 30s | "Extract entities from..." |
| creative_writing | 4-6 | 60s | "Write a blog post about..." |
| debugging | 4-6 | 90s | "Why does this code fail when..." |
| architecture | 6-8 | 120s | "Design a system that..." |
| documentation | 3-4 | 60s | "Document this API endpoint" |
| refactoring | 4-6 | 90s | "Simplify this function" |
| general | 3-4 | 60s | Anything that does not match above |

Categories with "4 + 4" workers (code_review, security_audit) use the two-panel adversarial review pattern: 4 workers for initial review and 4 separate workers for meta-review, with zero model overlap between panels.

The classifier is a rule-based system augmented with keyword matching and prompt structure analysis. It is deliberately simple -- misclassification costs some efficiency (wrong model selection) but does not cause failures, because all models can handle all task types with varying quality.

---

## Weighted Voting

Not all model responses receive equal weight in consensus synthesis. The arbiter scores each response on four dimensions before merging.

**Capability score.** A per-model, per-task-type quality score maintained by the routing system. Updated via exponentially weighted moving average (EWMA) after each execution. Range: 0.0 to 1.0.

**Confidence.** Self-reported confidence from the model's response, when available. Extracted via structured output or heuristic parsing. Models that hedge ("I'm not sure, but...") receive lower confidence weights. Range: 0.0 to 1.0.

**Historical accuracy.** Rolling accuracy rate for this model on this task category over the last 30 days. Computed from the `monitoring.execution_summary` table. Range: 0.0 to 1.0.

**Model tier.** A static tier assignment reflecting the model's general capability level. Tier 1 (frontier: Opus, GPT-4o, Gemini Ultra) receives a 1.0 multiplier. Tier 2 (strong: Sonnet, DeepSeek-V3, Qwen-Max) receives 0.85. Tier 3 (fast: Haiku, Gemini Flash, Groq-served) receives 0.7.

The final weight for a model's response is:

```
weight = capability_score * 0.4 + confidence * 0.2 + historical_accuracy * 0.3 + tier * 0.1
```

The arbiter receives all responses along with their computed weights and is instructed to give proportionally more consideration to higher-weighted responses during synthesis.

---

## Circuit Breaker Pattern

Each provider and each individual model endpoint maintains a circuit breaker to prevent cascading failures from unhealthy services.

```
    CIRCUIT BREAKER STATE MACHINE

    +--------+     error_count >= threshold     +--------+
    |        |  -------------------------------->        |
    | CLOSED |                                  |  OPEN  |
    |        | <--------------------------------|        |
    +--------+     (normal operation)           +--------+
        ^                                          |
        |                                          | cooldown expires
        |                                          | (default: 30s)
        |          +-----------+                   |
        +--------- |           | <-----------------+
          success  | HALF-OPEN |
                   |           | ---- failure ----> OPEN
                   +-----------+
```

**Closed (normal).** Requests pass through. Errors are counted. When the error count reaches the threshold (default: 5 consecutive errors), the breaker trips to Open.

**Open (tripped).** All requests to this provider/model are immediately rejected without making an API call. After the cooldown period (default: 30 seconds), the breaker transitions to Half-Open.

**Half-Open (testing).** A single probe request is allowed through. If it succeeds, the breaker returns to Closed and the error count resets. If it fails, the breaker returns to Open with a doubled cooldown (exponential backoff, capped at 5 minutes).

Circuit breaker state is stored in Redis for cross-process visibility. The `/api/circuit` blueprint exposes endpoints to inspect breaker state, manually trip a breaker (for maintenance), and manually reset a breaker.

---

## Rate Limiting

Rate limits are enforced per provider to respect API quotas and prevent abuse.

```
Provider          Requests/min    Tokens/min       Strategy
---------         ------------    ----------       --------
OpenRouter        60              unlimited        Token bucket (Redis)
Anthropic         50              100,000          Sliding window
Google            60              unlimited        Token bucket (Redis)
Groq              30              6,000            Sliding window
Cerebras          30              unlimited        Token bucket (Redis)
DeepSeek          60              unlimited        Token bucket (Redis)
```

Rate limiting uses Redis-backed token buckets implemented as atomic Lua scripts. When a provider's rate limit is reached, the orchestrator either queues the request (if the caller accepts async processing) or falls back to the next provider in the fallback chain.

Rate limit state is exposed via the `/api/providers` blueprint, which reports current usage, remaining quota, and estimated time until quota replenishment for each provider.

---

## Error Handling

The orchestration layer distinguishes between recoverable and non-recoverable errors.

**Recoverable (automatic retry):**
- Provider timeout (retry on different provider)
- Rate limit exceeded (wait and retry, or fall back)
- Worker SSH connection failure (retry on different worker)
- Model returns malformed response (retry same model once, then fall back)

**Non-recoverable (return error to caller):**
- All providers in fallback chain exhausted
- Task classification failure (malformed request)
- Authentication failure
- All fleet workers unreachable

Retries follow exponential backoff with jitter. The maximum retry count per request is 3 (configurable per task category). Each retry attempt is logged to `monitoring.execution_summary` for post-hoc analysis.
