# Consensus Engine

The consensus engine queries multiple LLMs for the same task and synthesizes their responses into a single, higher-quality answer. It is the core differentiator of the orchestration framework: instead of trusting one model, the system triangulates across models with different training data, architectures, and failure modes.

---

## How Consensus Works

A consensus request proceeds through four stages: dispatch, collection, scoring, and synthesis.

```
                          CONSENSUS FLOW

  Task: "Review this function for off-by-one errors"
    |
    v
  +------------------+
  | Model Selection   |    Thompson Sampling selects 6 models
  | (Thompson Sampling)|    from 200+ candidates based on
  +------------------+    task type: code_review
    |
    |  Dispatch in parallel
    v
  +----------+  +----------+  +----------+  +----------+  +----------+  +----------+
  | Worker 1 |  | Worker 2 |  | Worker 3 |  | Worker 4 |  | Worker 5 |  | Worker 6 |
  | Sonnet   |  | Opus     |  | GPT-4o   |  | Gemini   |  | DeepSeek |  | Qwen     |
  +----------+  +----------+  +----------+  +----------+  +----------+  +----------+
    |  resp A     |  resp B     |  resp C     |  timeout    |  resp E     |  resp F
    |             |             |             |  (skip)     |             |
    v             v             v                           v             v
  +------------------------------------------------------------------------+
  |                         ARBITER (Fable)                                 |
  |                                                                        |
  |  Inputs:                                                               |
  |    - 5 worker responses (A, B, C, E, F)                                |
  |    - Weighted scores per response                                      |
  |    - Original task description                                         |
  |                                                                        |
  |  Process:                                                              |
  |    1. Score each response (relevance, correctness, completeness)        |
  |    2. Identify agreement points (mentioned by 3+ responses)            |
  |    3. Identify disagreement points (strong contradiction)              |
  |    4. Weight by capability score and historical accuracy                |
  |    5. Synthesize unified answer                                        |
  |    6. Flag unresolved disagreements                                    |
  |                                                                        |
  |  Output:                                                               |
  |    - Synthesized consensus response                                    |
  |    - Per-response scores                                               |
  |    - Agreement/disagreement summary                                    |
  |    - Confidence level (high/medium/low)                                |
  +------------------------------------------------------------------------+
    |
    v
  Consensus Response returned to caller
```

---

## Consensus Strategies

The system supports five consensus strategies, selectable per request or per task category.

### Rotating (default)

The arbiter model rotates through a predefined list across consecutive requests. This prevents arbiter bias from accumulating: if Opus is the arbiter, it may subtly favor Opus-like reasoning patterns. Rotation distributes this bias across models.

Rotation order is deterministic (round-robin keyed by request count mod pool size), so the same request retried immediately will get the same arbiter. The rotation pool is configurable and defaults to: Opus, Sonnet, Fable, Haiku.

### Single

A fixed arbiter model handles all synthesis. Useful when you want consistent synthesis behavior (e.g., always use Opus for architecture reviews). Faster than rotating because the arbiter's context window and prompt template can be pre-warmed.

### Majority

No arbiter. The system counts agreement among worker responses using semantic similarity. If 4 of 6 workers agree on a finding, it is included in the consensus. Findings with fewer than majority support are discarded.

Best for classification tasks, binary decisions, and factual lookups where there is a clear right answer. Poor for open-ended generation where responses are structurally different but equally valid.

### Weighted

Similar to majority but responses are weighted by capability score, historical accuracy, and model tier before counting. A high-weight model's agreement counts more than a low-weight model's. This allows the system to defer to stronger models when there is disagreement.

### Pairwise

Each pair of worker responses is compared head-to-head by a judge model. The response that wins the most pairwise comparisons becomes the basis for the consensus. This is the most expensive strategy (N*(N-1)/2 comparisons for N responses) but produces the most defensible ranking.

Used sparingly, primarily for evaluating model quality during GA optimization runs where accurate ranking is worth the additional cost.

---

## The Arbiter

The arbiter is a model tasked with synthesizing multiple worker responses into a single coherent answer. It is distinct from the worker models and receives a structured prompt containing:

1. The original task description
2. All worker responses, labeled by model name
3. Computed weight for each response
4. Instructions for synthesis

The arbiter prompt instructs the model to:

- Read all responses before forming an opinion
- Give proportionally more weight to higher-scored responses
- Identify points where 3+ responses agree (high confidence)
- Identify points where responses contradict each other (flag for review)
- Produce a unified answer that incorporates the strongest elements from each response
- Note any findings that only one model raised but that appear credible (minority insights)
- Assign an overall confidence level: high (strong agreement), medium (partial agreement), low (significant disagreement)

The arbiter's output is structured:

```json
{
  "consensus": "The synthesized answer...",
  "confidence": "high",
  "agreement_points": [
    "All models agree the loop terminates one iteration early"
  ],
  "disagreement_points": [
    "Models disagree on whether the fix should use <= or < + 1"
  ],
  "minority_insights": [
    "Only DeepSeek noted the potential integer overflow on line 42"
  ],
  "per_model_scores": {
    "sonnet": 0.85,
    "opus": 0.92,
    "gpt-4o": 0.78,
    "deepseek": 0.88,
    "qwen": 0.74
  }
}
```

---

## Consensus Caching

Identical consensus queries are cached to avoid redundant multi-model invocations.

**Cache key.** SHA-256 hash of the normalized task description, model list, and strategy. Normalization strips whitespace variations and sorts model lists to ensure equivalent requests produce the same key.

**Cache storage.** Redis, with a configurable TTL (default: 5 minutes for code tasks, 30 minutes for research, 60 minutes for documentation).

**Cache invalidation.** Automatic on TTL expiry. Manual invalidation via the `/api/cache` endpoint. The cache is also invalidated when Thompson Sampling parameters change significantly (indicating the routing landscape has shifted).

**Cache hit rate.** In practice, cache hit rates are low (5-15%) because most consensus queries are unique. The cache primarily benefits retry scenarios and workflow steps that re-query the same information.

---

## Disagreement Detection

When models strongly disagree, the consensus engine flags the disagreement rather than silently picking a winner. Disagreement is detected by semantic similarity analysis of worker responses.

**Detection method.** Each pair of worker responses is compared using cosine similarity on their embeddings (384-dimensional, computed via sentence-transformers). If any pair has a similarity score below 0.3, the system flags a strong disagreement.

**Disagreement handling.** The disagreement is surfaced in the consensus response as a `disagreement_points` array. The arbiter is instructed to explicitly describe both sides of the disagreement rather than arbitrarily choosing one.

**Escalation.** For task categories marked as high-stakes (security_audit, architecture), disagreements with similarity below 0.2 trigger an automatic escalation: the system re-runs the consensus with a larger worker pool (8 models instead of 4) to gather more opinions.

---

## Default Worker Pool

The default worker pool for general consensus queries includes models selected for diversity of training data and architecture.

| Model | Provider | Strength | Typical Role |
|-------|----------|----------|--------------|
| claude-sonnet | Anthropic | Balanced reasoning, strong code | Primary worker |
| claude-opus | Anthropic | Deep analysis, nuanced reasoning | Primary worker |
| claude-haiku | Anthropic | Fast, cost-effective, good at classification | Fast-path worker |
| gpt-4o | OpenRouter | Strong structured output, broad knowledge | Primary worker |
| gemini-2.0-flash | Google | Long context, fast inference | Long-context worker |
| llama-3.3-70b | Groq | Fast inference via LPU, open-weight reasoning | Speed worker |

For specialized task categories, the worker pool is adjusted. Code review adds DeepSeek-V3 and Qwen3-Coder (strong code-specific models). Research adds Hermes-405B and Nemotron-Ultra (strong at long-form synthesis). Security audit adds models with specific security training.

Worker pool composition is subject to GA optimization (see `tools/ga_team_selection_fixed.py`), which evolves team compositions based on consensus quality metrics.

---

## When Consensus Fails

Consensus can fail for several reasons. Each failure mode has a defined fallback.

**All workers time out.** No responses collected. The system retries with a longer timeout (2x default) on a different set of workers. If the retry also fails, it returns an error to the caller with the specific timeout details.

**Only one worker responds.** Insufficient data for consensus. The system returns the single response with a `confidence: "low"` flag and a note that consensus was not achieved. The caller can decide whether to accept the single-model answer.

**Arbiter fails.** The synthesis step itself fails (arbiter model unavailable or returns malformed output). The system falls back to the highest-weighted worker response as the consensus result, with a note that arbiter synthesis was not performed.

**Extreme disagreement.** All responses have pairwise similarity below 0.2. The system returns all individual responses to the caller without synthesis, flagged as `consensus: "none"`. This typically indicates an ambiguous task that needs clarification.

```
                     CONSENSUS FAILURE FALLBACK CHAIN

  Normal:       6 workers --> arbiter synthesis --> consensus response
                                    |
                              arbiter fails?
                                    |
  Fallback 1:   6 workers --> highest-weighted response (no synthesis)
                    |
              only 1 response?
                    |
  Fallback 2:   1 worker --> single response, confidence: low
                    |
              0 responses?
                    |
  Fallback 3:   retry with 2x timeout, different workers
                    |
              retry also fails?
                    |
  Error:        return error to caller
```

---

## Observability

Every consensus execution produces telemetry stored in PostgreSQL for analysis and debugging.

**Execution record** (`workflow.executions`): task description, models used, strategy, arbiter, total duration, outcome (success/failure/partial).

**Per-worker results** (`workflow.worker_results`): model name, response time, response length, success/failure, error details if failed.

**Arbiter decision** (`workflow.arbiter_decisions`): per-model scores, agreement points, disagreement points, confidence level, synthesis duration.

**Thompson Sampling update** (`learning.strategy_performance`): which model was selected, whether the selection was exploratory or exploitative, outcome for bandit parameter update.

This telemetry enables post-hoc analysis of consensus quality, identification of systematically underperforming models, and calibration of the weighted voting parameters.
