# Thompson Sampling

Thompson Sampling is the primary multi-armed bandit algorithm used for
real-time strategy selection across the orchestration framework. It governs
which model, prompt template, and orchestration pattern to use for each
incoming task, balancing the need to exploit known-good strategies against the
need to explore potentially better alternatives.

---

## The Explore/Exploit Tradeoff

Every time the system routes a task, it faces a decision:

- **Exploit:** Use the strategy with the best observed performance. This
  maximizes short-term quality but may miss a better option that has not been
  tried enough times.
- **Explore:** Try a less-tested strategy. This sacrifices short-term
  performance for information that may improve long-term outcomes.

A naive approach (always exploit, or explore with fixed probability) either
converges prematurely or wastes resources on strategies that have already been
proven inferior. Thompson Sampling solves this by exploring in proportion to
each strategy's probability of being optimal.

---

## How It Works

### Bayesian State

Each strategy arm maintains a Beta distribution parameterized by:

```
alpha = successes + 1
beta  = failures + 1
```

The `+1` terms are the prior (a uniform Beta(1,1) distribution), expressing
no initial preference. As outcomes accumulate, the distribution concentrates
around the strategy's true success rate.

```
Strategy A:  15 successes, 3 failures  -> Beta(16, 4)   mean = 0.80
Strategy B:  4 successes, 2 failures   -> Beta(5, 3)    mean = 0.625
Strategy C:  1 success, 0 failures     -> Beta(2, 1)    mean = 0.67
```

### Selection Process

At decision time, the algorithm:

1. Samples one random value from each strategy's Beta distribution.
2. Selects the strategy whose sample is highest.
3. Executes the task with that strategy.
4. Updates the selected strategy's alpha or beta based on the outcome.

```
Step 1: Sample from each distribution
  A ~ Beta(16, 4)  -> sample = 0.78
  B ~ Beta(5, 3)   -> sample = 0.71
  C ~ Beta(2, 1)   -> sample = 0.85   <-- highest

Step 2: Select C (even though A has higher mean)

Step 3: Execute with strategy C

Step 4: C succeeds -> C becomes Beta(3, 1), mean = 0.75
   or:  C fails    -> C becomes Beta(2, 2), mean = 0.50
```

Strategy C was selected despite having fewer observations than A. Its wide
distribution (low confidence) produced a high sample, triggering exploration.
After enough trials, C's distribution will narrow and its selection rate will
reflect its true quality.

### Convergence Behavior

```
Early (few observations):   Wide distributions, frequent exploration
                            +-------+-------+-------+
                         A: |  ***********          |
                         B: |    *************      |
                         C: |  ****************    |
                            +-------+-------+-------+
                            0.0     0.5     1.0

Late (many observations):   Narrow distributions, mostly exploitation
                            +-------+-------+-------+
                         A: |           ***         |
                         B: |        **             |
                         C: |             ****      |
                            +-------+-------+-------+
                            0.0     0.5     1.0
```

---

## Why Thompson Sampling

Three alternative bandit algorithms are available in the system. Thompson
Sampling is the default for the following reasons:

```
+-------------------+--------------------+-----------------------------------+
| Algorithm         | Exploration Method | Limitation                        |
+-------------------+--------------------+-----------------------------------+
| Epsilon-greedy    | Random with fixed  | Requires tuning epsilon. Too high |
|                   | probability        | wastes resources; too low misses  |
|                   |                    | good strategies.                  |
+-------------------+--------------------+-----------------------------------+
| UCB1              | Optimism: adds     | Deterministic. Explores each arm  |
|                   | confidence bonus   | equally regardless of observed    |
|                   | to mean            | performance. Over-explores weak   |
|                   |                    | arms before abandoning them.      |
+-------------------+--------------------+-----------------------------------+
| Exp3              | Adversarial: no    | Designed for adversarial rewards. |
|                   | stochastic         | Overly conservative when rewards  |
|                   | assumptions        | are stochastic (most of ours).    |
+-------------------+--------------------+-----------------------------------+
| Thompson Sampling | Bayesian: samples  | Exploration decreases naturally   |
|                   | from posterior      | as confidence grows. No tunable   |
|                   |                    | parameters. Handles non-stationary|
|                   |                    | rewards well.                     |
+-------------------+--------------------+-----------------------------------+
```

Thompson Sampling's key advantage is that it requires no hyperparameter
tuning. Epsilon-greedy needs an epsilon value (and often an annealing
schedule). UCB1 needs a confidence multiplier. Thompson Sampling derives its
exploration rate directly from the data.

---

## Strategy Arms

A strategy arm is defined as a unique combination of:

| Dimension              | Examples                                       |
|------------------------|-------------------------------------------------|
| Prompt template        | concise, detailed, step-by-step, adversarial   |
| Orchestration pattern  | parallel-consensus, serial-refine, round-robin  |
| Verification method    | self-verify, cross-model-verify, human-in-loop  |
| Model                  | opus, sonnet, deepseek-chat, qwen3-coder        |
| Task type              | code-review, research, summarization, routing    |

Not all combinations are valid. The system registers arms explicitly rather
than enumerating the full combinatorial space. New arms are created through
two channels:

1. **Manual registration:** An operator defines a new strategy configuration
   and registers it via the REST API.
2. **GA evolution:** The genetic algorithm optimizer evolves new configurations
   (see [Genetic Algorithms](https://github.com/FlossWare/.github/blob/main/docs/learning/genetic_algorithms.md)) and registers the fittest
   individuals as new bandit arms.

---

## Integration with Genetic Algorithms

Thompson Sampling and GA optimization operate at different timescales:

```
+-----------------------+------------------------------------------+
| Thompson Sampling     | Genetic Algorithm                        |
+-----------------------+------------------------------------------+
| Per-request decisions | Generational evolution (hours/days)      |
| Selects from existing | Creates new strategy configurations     |
| arms                  |                                          |
| Updates alpha/beta    | Evaluates fitness across populations    |
| after each execution  |                                          |
+-----------------------+------------------------------------------+

                    GA evolves new configs
                           |
                           v
              +------------------------+
              | Register as new bandit |
              | arm with Beta(1, 1)    |
              +------------------------+
                           |
                           v
              Thompson Sampling includes
              new arm in selection pool
                           |
                           v
              Arm accumulates observations,
              distribution narrows
```

GA finds globally promising configurations through population-based search.
Thompson Sampling tests them in production traffic and allocates real requests
in proportion to observed quality.

---

## Performance Tracking

Strategy performance is stored in `learning.strategy_performance`:

```sql
SELECT strategy_id, prompt_template, orchestration_pattern,
       verification_method, alpha, beta,
       alpha::float / (alpha + beta) AS estimated_success_rate,
       (alpha + beta - 2) AS total_observations
FROM learning.strategy_performance
ORDER BY alpha::float / (alpha + beta) DESC;
```

Multi-dimensional tracking allows analysis along any axis:

```sql
-- Best prompt template regardless of other dimensions
SELECT prompt_template,
       SUM(alpha - 1) AS total_successes,
       SUM(beta - 1) AS total_failures,
       SUM(alpha - 1)::float / SUM(alpha + beta - 2) AS success_rate
FROM learning.strategy_performance
GROUP BY prompt_template
ORDER BY success_rate DESC;
```

---

## Lifecycle of a Strategy

```
1. CREATED       New arm registered (manually or via GA)
                 State: Beta(1, 1), no observations

2. EXPLORING     Thompson Sampling occasionally selects this arm
                 due to its wide distribution
                 State: Beta(low alpha, low beta)

3. CONVERGING    Distribution narrows as observations accumulate
                 Selection frequency reflects true quality
                 State: Beta(moderate alpha, moderate beta)

4. EXPLOITED     High-quality arm selected frequently
                 or
   ABANDONED     Low-quality arm rarely selected (but never
                 fully excluded -- its distribution always has
                 some probability mass above zero)

5. RETIRED       Arm removed from the pool if it has >100
                 observations and a success rate below 20%
                 (configurable threshold)
```

Retirement is the only non-Bayesian intervention. Without it, the arm pool
grows monotonically as the GA produces new candidates. The retirement
threshold is deliberately low (20%) to avoid prematurely discarding arms
that may perform well for specific task types not yet observed.
