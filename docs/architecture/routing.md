# Model Routing

The routing subsystem decides which models handle each task. It maintains a real-time capability model for every available LLM and uses Thompson Sampling to balance exploitation of proven models with exploration of alternatives. Since all 200+ models in the pool are accessed via free API tiers, routing optimizes purely for quality and latency -- cost is not a factor.

---

## Task-Aware Routing

Routing begins with task classification. The classifier assigns each incoming request to one of 15 categories (see the Orchestration Layer document for the full list). The routing system maintains a separate capability profile for every model-category pair, so a model's strength at code generation does not influence its selection for research tasks.

```
                        ROUTING DECISION FLOW

  Incoming task: "Find security vulnerabilities in this Go code"
    |
    v
  +---------------------+
  | Task Classification  |
  | --> security_audit   |
  +---------------------+
    |
    v
  +---------------------+
  | Filter: Active models|    Remove models with tripped circuit breakers,
  | for security_audit   |    rate-limited providers, or known incompatible
  +---------------------+    context window sizes.
    |
    v
  +---------------------+
  | Thompson Sampling    |    For each eligible model, sample from its
  | (per-model Beta      |    Beta(alpha, beta) distribution for the
  |  distribution)       |    security_audit category.
  +---------------------+
    |
    v
  +---------------------+
  | Select top N models  |    Pick the N models with the highest samples.
  | (N = worker count    |    N is determined by the task category's
  |  for this category)  |    configured worker count (e.g., 4+4 for
  +---------------------+    security_audit).
    |
    v
  Dispatch to fleet workers
```

---

## Dynamic Capability Scores

Each model-category pair has a capability score maintained as an exponentially weighted moving average (EWMA) of execution outcomes.

**Update formula:**

```
score_new = alpha * outcome + (1 - alpha) * score_old
```

Where `alpha = 0.1` (recent outcomes contribute 10% to the new score), and `outcome` is a normalized quality measure (0.0 to 1.0) derived from:

- **Arbiter scoring.** The arbiter assigns a per-model score during consensus synthesis. This is the primary signal.
- **Consensus agreement.** How often the model's response aligned with the final consensus. Models that consistently disagree with consensus receive lower scores (though a model that disagrees and is right receives a bonus when the disagreement is later validated).
- **Response validity.** Whether the response was parseable, relevant, and within the expected format. Malformed or off-topic responses receive a 0.0 outcome.
- **Latency.** Normalized inverse of response time. Faster responses receive a slight bonus (up to 0.05) to break ties between equally capable models.

Capability scores are stored in the `learning.strategy_performance` table and are updated after every consensus execution.

---

## Thompson Sampling

Thompson Sampling is a Bayesian approach to the multi-armed bandit problem. In this system, each model-category pair is an "arm" with an unknown reward probability. The algorithm maintains a Beta distribution for each arm, parameterized by observed successes (alpha) and failures (beta).

**Selection process:**

1. For each eligible model `m` in category `c`, sample a value from `Beta(alpha_mc, beta_mc)`.
2. Rank models by their sampled values.
3. Select the top N models.

**Why Thompson Sampling works well here:**

- **Natural exploration.** A model with few observations has a wide Beta distribution. Its samples will occasionally be high, causing it to be selected and evaluated. As observations accumulate, the distribution narrows and the model is selected at a rate proportional to its true quality.
- **Automatic deprioritization.** A model that consistently fails accumulates beta (failure) counts. Its distribution shifts left, making high samples increasingly unlikely. The model is effectively deprioritized without being permanently removed.
- **Cold-start handling.** New models are initialized with `Beta(1, 1)` (uniform prior), giving them equal initial probability. Family-based priors can optionally warm-start a model: a new Claude variant inherits the Claude family's aggregate alpha/beta as its starting point.

**Parameter updates:**

After each consensus execution, for each participating model:
- If the model's response was scored above the category median: `alpha += 1`
- If scored below the category median: `beta += 1`
- If the response timed out or was malformed: `beta += 2` (stronger penalty)

Parameters are stored in the `learning.strategy_performance` table. The current state of all arms can be inspected via `GET /api/routing/state`.

---

## Fallback Chains

When the primary selected model is unavailable (provider outage, rate limit, circuit breaker tripped), the routing system follows a fallback chain.

Each model has a pre-configured fallback list ordered by capability similarity:

```
Primary             Fallback 1              Fallback 2              Fallback 3
-------             ----------              ----------              ----------
claude-opus     --> deepseek-v3         --> qwen-max            --> gpt-4o
claude-sonnet   --> deepseek-chat       --> gemini-2.0-flash    --> llama-3.3-70b
claude-haiku    --> gemini-flash        --> llama-3.1-8b        --> qwen-turbo
gpt-4o          --> claude-opus         --> deepseek-v3         --> gemini-pro
gemini-pro      --> claude-sonnet       --> gpt-4o              --> deepseek-v3
deepseek-v3     --> claude-opus         --> qwen-max            --> gpt-4o
```

Fallback chains are configured to maximize provider diversity. If Anthropic is down, Claude models fall back to DeepSeek, Qwen, or OpenAI -- never to another Anthropic model. This ensures that a single provider outage does not cascade to eliminate multiple fallback levels.

The system currently maintains 78+ provider endpoints across its fallback chains, making complete model unavailability extremely unlikely.

---

## Model Capability Matrix

The routing system maintains a capability matrix: a per-model, per-category table of quality and latency scores.

```
Model              code_gen  code_review  research  math  security  summary
-----              --------  -----------  --------  ----  --------  -------
claude-opus         0.94      0.96         0.93     0.91   0.95      0.88
claude-sonnet       0.91      0.93         0.90     0.87   0.92      0.90
claude-haiku        0.78      0.80         0.75     0.72   0.77      0.85
gpt-4o              0.89      0.88         0.91     0.90   0.86      0.87
gemini-2.0-flash    0.82      0.79         0.85     0.83   0.78      0.86
deepseek-v3         0.90      0.92         0.86     0.88   0.89      0.82
qwen-max            0.87      0.85         0.84     0.86   0.83      0.81
llama-3.3-70b       0.83      0.81         0.82     0.80   0.79      0.80
hermes-405b         0.85      0.84         0.89     0.82   0.84      0.83
nemotron-ultra      0.86      0.87         0.88     0.84   0.86      0.82
```

*(Scores shown are illustrative. Actual scores are continuously updated via EWMA from execution data.)*

This matrix is not static. It is recomputed from the Thompson Sampling state and EWMA scores every time the routing endpoint is queried. The `/api/routing/matrix` endpoint returns the current snapshot for inspection.

---

## How New Models Enter the Pool

When a new model becomes available (e.g., a provider releases a new version), it enters the routing pool through a defined process.

1. **Registration.** The model is added to the model registry via `POST /api/models/register` with its provider, context window size, and model family.

2. **Prior initialization.** If the model belongs to a known family (Claude, GPT, Gemini, DeepSeek, etc.), its Thompson Sampling parameters are initialized from the family's aggregate performance: `Beta(family_alpha_avg, family_beta_avg)`. If the model is from an unknown family, it starts with `Beta(1, 1)` (uniform prior).

3. **Exploration phase.** Thompson Sampling's natural exploration ensures the new model receives trial assignments. With a uniform prior, a new model has roughly a 50% chance of being sampled above the median on any given draw, so it will be tried within a few consensus rounds.

4. **Evaluation.** Over the first 20-50 tasks, the model accumulates execution data. Its capability scores converge, and Thompson Sampling narrows its distribution. By 50 tasks, the system has a reliable estimate of the model's quality per category.

5. **Steady state.** The model is selected at a rate proportional to its quality. High-quality models are selected frequently; low-quality models are selected rarely (but never zero -- Thompson Sampling always maintains a non-zero exploration probability).

No manual intervention is required at any step. The system autonomously evaluates and positions new models.

---

## How Underperforming Models Get Deprioritized

A model that consistently produces low-quality responses is deprioritized through two mechanisms.

**Thompson Sampling decay.** Accumulated beta (failure) counts shift the model's distribution left. After 20 failures with only 2 successes, the model's distribution is `Beta(3, 21)`, which has a mean of 0.125 and a 95th percentile of approximately 0.28. This model will almost never be sampled above a model with `Beta(15, 5)` (mean 0.75), effectively removing it from selection without a hard ban.

**Circuit breaker.** If a model produces 5 consecutive errors (timeouts, malformed responses, API errors), its circuit breaker trips and it is excluded from the selection pool entirely for the cooldown period. Repeated trips extend the cooldown via exponential backoff.

**Manual override.** The `/api/routing/override` endpoint allows administrators to manually exclude a model (temporary or permanent), pin a model to specific task categories, or reset a model's Thompson Sampling parameters to trigger re-evaluation.

**No permanent bans.** The system never permanently removes a model from the pool. Even a model with `Beta(1, 100)` has a non-zero probability of being sampled. This is deliberate: providers sometimes fix issues with their models, and Thompson Sampling will detect the improvement through exploration.

---

## Quality-First Routing

A key property of this system is that cost is irrelevant to routing decisions. All 200+ models in the pool are accessed via free API tiers (OpenRouter free models, Anthropic free tier, Google free tier, Groq free tier, etc.). This eliminates the quality-cost tradeoff that dominates most LLM routing systems.

The practical effect: the routing system always selects the highest-quality models for the task. There is no reason to route a task to a weaker model to save money. This simplifies the routing algorithm (one objective instead of a Pareto frontier) and means the system automatically uses the best available models at all times.

If the system were to add paid-tier models in the future, the routing algorithm would need to be extended with a cost-quality tradeoff. This would involve adding a cost dimension to the capability matrix and allowing callers to specify a cost constraint (e.g., "best quality under $0.01 per request"). Thompson Sampling would be augmented with a cost-aware reward function. The current architecture anticipates this extension but does not implement it.
