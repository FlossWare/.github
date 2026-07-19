# Genetic Algorithm Evolution

The orchestration framework uses genetic algorithms (GAs) to evolve system
configurations over time. Configurations are encoded as chromosomes, execution
history provides the fitness landscape, and standard evolutionary operators
(selection, crossover, mutation) search the space for higher-performing
configurations across seven optimization domains.

---

## Core Concept

The GA treats every tunable aspect of the system as a gene. A complete
configuration is a chromosome. A population of chromosomes is evaluated against
real execution data, and the fittest individuals are recombined and mutated to
produce the next generation.

```
Generation N                        Generation N+1
+--------+--------+--------+       +--------+--------+--------+
| Config | Config | Config |  -->  | Config | Config | Config |
|   A    |   B    |   C    |       |   A'   |   D    |   B'   |
| fit=72 | fit=85 | fit=61 |       | fit=88 | fit=79 | fit=83 |
+--------+--------+--------+       +--------+--------+--------+
     |        |                          ^        ^
     |        +--- survives (elite) -----+        |
     |                                            |
     +------------ crossover(A, B) ---------------+
```

This is not simulated. Fitness scores are computed from actual workflow
executions stored in the PostgreSQL database. The GA queries execution history,
computes fitness, and evolves configurations that demonstrably improve
real-world outcomes.

---

## The GA Loop

Each optimizer follows the same evolutionary loop:

```
1. INITIALIZE    Create initial population (random or seeded from
                 current production configs)
                      |
                      v
2. EVALUATE      Compute fitness for each chromosome using
                 real execution data from the REST API
                      |
                      v
3. SELECT        Tournament selection: pick k random individuals,
                 take the fittest. Repeat to fill mating pool.
                      |
                      v
4. CROSSOVER     Uniform crossover: for each gene, randomly choose
                 from parent A or parent B
                      |
                      v
5. MUTATE        With probability p, replace a gene with a random
                 valid value from its allowed range
                      |
                      v
6. ELITISM       Copy the top e% of the current generation into
                 the next generation unchanged
                      |
                      v
7. REPEAT        Go to step 2 with the new population
```

### Core Data Structure

The base chromosome is a `FleetChromosome` dataclass with six genes common
to all optimizers:

```python
@dataclass
class FleetChromosome:
    workers: int           # Number of parallel workers (1-8)
    arbiter: str           # Model used for final synthesis
    timeout: int           # Per-worker timeout in seconds
    consensus_type: str    # "majority", "weighted", "unanimous"
    retry_limit: int       # Max retries on failure (0-5)
    diversity_weight: float # 0.0-1.0, weight given to model diversity
```

Each optimizer extends this with domain-specific genes.

---

## The Seven Optimizers

### 1. Model Routing Optimizer

**Evolves:** Mappings from task types to preferred models.

**Chromosome:** A dictionary of `{task_type: model_id}` entries, plus routing
parameters (fallback chains, load thresholds, cost caps).

**Fitness function:**

```
fitness = quality * 0.6 - cost * 0.2 - latency * 0.2
```

- `quality`: Average quality score from arbiter evaluations (0-1)
- `cost`: Normalized API cost per request
- `latency`: Normalized p95 response time

**Data source:** `monitoring.execution_summary` joined with `workflow.feedback`

**Example chromosome:**

```python
{
    "code_review":   "deepseek-chat",
    "summarization": "gemini-2.0-flash",
    "research":      "claude-sonnet-4",
    "routing":       "haiku",
    "fallback_chain": ["openrouter", "groq", "cerebras"],
    "cost_cap_usd":  0.05
}
```

### 2. RAG Retrieval Optimizer

**Evolves:** Chunking and retrieval parameters for the knowledge base.

**Chromosome:**

| Gene                | Range         | Effect                            |
|---------------------|---------------|-----------------------------------|
| chunk_size          | 128-2048      | Characters per text chunk         |
| chunk_overlap       | 0-256         | Overlap between adjacent chunks   |
| top_k               | 3-50          | Number of results to retrieve     |
| similarity_threshold| 0.3-0.9       | Minimum cosine similarity         |
| category_boosts     | dict          | Per-category relevance multipliers|

**Fitness function:**

```
fitness = relevance + coverage + efficiency + diversity
```

- `relevance`: Precision of retrieved chunks (rated by arbiter)
- `coverage`: Recall of relevant information
- `efficiency`: Inverse of total tokens retrieved (penalizes over-retrieval)
- `diversity`: Source diversity among retrieved chunks

### 3. Team Composition Optimizer

**Evolves:** Which models to include in review and meta-review panels.

**Chromosome:** Two lists (review panel, meta-review panel) with a zero-overlap
constraint enforced during crossover and mutation.

**Fitness function:**

```
fitness = quality + diversity - cost
```

- `quality`: Average quality score from panel outputs
- `diversity`: Average pairwise cosine distance between model outputs
  (higher = more diverse perspectives)
- `cost`: Total API cost for the panel

**Constraint:** If crossover or mutation produces a chromosome where a model
appears in both panels, the duplicate is replaced with a random model from the
unused pool.

### 4. Training Data Curation Optimizer

**Evolves:** How training examples are selected and weighted for downstream
evaluation datasets.

**Chromosome:**

| Gene                | Range         | Effect                             |
|---------------------|---------------|------------------------------------|
| category_weights    | dict          | Weight per document category       |
| quality_threshold   | 0.1-0.9       | Minimum quality score to include   |
| dedup_threshold     | 0.7-0.99      | Similarity threshold for dedup     |
| max_examples_per_cat| 10-1000       | Cap per category                   |

**Fitness function:** Downstream evaluation score of models fine-tuned (or
prompted) with the curated dataset.

### 5. Prompt Template Optimizer

**Evolves:** Prompt structure, instruction phrasing, and example selection.

**Chromosome:**

| Gene                | Values                                       |
|---------------------|----------------------------------------------|
| structure           | "direct", "chain-of-thought", "few-shot"     |
| instruction_style   | "imperative", "descriptive", "socratic"      |
| example_count       | 0-5                                          |
| example_selection   | "random", "similar", "diverse", "hardest"    |
| system_prompt_length| "minimal", "standard", "detailed"            |

**Fitness function:** Response quality as rated by the evaluation harness.

### 6. Adversarial Verification Optimizer

**Evolves:** Configuration of the adversarial verification panel.

**Chromosome:**

| Gene                | Range         | Effect                             |
|---------------------|---------------|------------------------------------|
| vote_threshold      | 0.5-0.9       | Fraction of votes to accept        |
| panel_size          | 3-7           | Number of verifier models          |
| panel_composition   | list          | Which models serve as verifiers    |
| challenge_depth     | 1-3           | Rounds of adversarial challenge    |

**Fitness function:**

```
fitness = true_positive_rate * 0.5 + (1 - false_positive_rate) * 0.5
```

Maximizes correct acceptance of valid outputs while minimizing incorrect
acceptance of flawed outputs. Ground truth comes from human-labeled evaluation
sets.

### 7. Workflow Configuration Optimizer

**Evolves:** Workflow execution parameters.

**Chromosome:**

| Gene                | Range / Values                                |
|---------------------|-----------------------------------------------|
| pattern             | "parallel", "serial", "map-reduce", "pipeline"|
| batch_size          | 1-8                                           |
| worker_count        | 1-8                                           |
| timeout_seconds     | 30-600                                        |
| retry_limit         | 0-5                                           |
| consensus_type      | "majority", "weighted", "unanimous"           |

**Fitness function:**

```
fitness = success * 0.4 + quality * 0.3 + efficiency * 0.2 + diversity * 0.1
```

- `success`: Binary success rate across executions
- `quality`: Arbiter quality scores
- `efficiency`: Inverse of resource consumption (tokens, time, cost)
- `diversity`: Model diversity in worker assignments

---

## Evolutionary Operators

### Tournament Selection

For each slot in the mating pool, `k` individuals are drawn at random from the
population. The fittest among them is selected. Tournament size `k` controls
selection pressure: larger `k` favors exploitation, smaller `k` favors
exploration.

```
Population:  [A=72, B=85, C=61, D=79, E=68, F=91, ...]
Tournament (k=3): Draw {C=61, B=85, E=68} -> Select B (fitness 85)
Tournament (k=3): Draw {A=72, F=91, D=79} -> Select F (fitness 91)
```

### Uniform Crossover

For each gene position, the offspring inherits from parent A or parent B with
equal probability:

```
Parent A: [workers=4, arbiter=opus,   timeout=120, consensus=majority]
Parent B: [workers=6, arbiter=sonnet, timeout=60,  consensus=weighted]
Mask:     [    A,          B,             A,             B           ]
Child:    [workers=4, arbiter=sonnet, timeout=120, consensus=weighted]
```

### Mutation

Each gene has an independent probability `p` of being replaced with a random
valid value:

```
Before: [workers=4, arbiter=sonnet, timeout=120, consensus=weighted]
Mutate: [    no,        no,          YES->90,        no            ]
After:  [workers=4, arbiter=sonnet, timeout=90,  consensus=weighted]
```

Mutation rates are set per-optimizer, typically 10-15%.

---

## Hyperparameters

| Parameter         | Typical Value | Notes                                |
|-------------------|---------------|--------------------------------------|
| Population size   | 20-30         | Larger for higher-dimensional spaces |
| Mutation rate     | 10-15%        | Per-gene probability                 |
| Elitism           | 10-50%        | Top individuals carried forward      |
| Tournament size   | 3             | Selection pressure control           |
| Generations       | 20-50         | Per evolution run                    |
| Convergence check | 5 generations | Stop if best fitness unchanged       |

Elitism ranges from 10% (aggressive exploration) to 50% (conservative, for
production-critical optimizers). Higher elitism preserves known-good configs
at the cost of slower exploration.

---

## Real Data, Not Simulation

All fitness evaluations query real execution history through the REST API.
No synthetic benchmarks or simulated environments are used.

```
+------------------+       +-----------+       +------------+
| GA Optimizer     | ----> | REST API  | ----> | PostgreSQL |
| (compute fitness)|       | /api/...  |       | monitoring |
+------------------+       +-----------+       | workflow   |
                                               | learning   |
                                               +------------+

Fitness for chromosome X:
  1. Query execution_summary WHERE config matches X's genes
  2. Join with workflow.feedback for quality scores
  3. Join with costs.entries for cost data
  4. Compute weighted fitness score
  5. Return to GA for selection
```

If a chromosome has no matching execution history (a novel configuration), its
fitness is estimated from the nearest neighbors in the configuration space,
weighted by distance. This avoids the cold-start problem without requiring
speculative execution.

---

## GA + Thompson Sampling Integration

The two systems operate at different timescales and serve complementary roles:

```
+-------------------------------------------+
|        Genetic Algorithm (offline)        |
|  - Searches configuration space           |
|  - Runs every N hours or on-demand        |
|  - Produces candidate configurations      |
|  - Population-level optimization          |
+-------------------------------------------+
              |
              | Register fittest as new bandit arms
              v
+-------------------------------------------+
|     Thompson Sampling (online)            |
|  - Selects per-request strategy           |
|  - Runs on every incoming task            |
|  - Balances explore/exploit in real-time  |
|  - Individual-level selection             |
+-------------------------------------------+
              |
              | Execution outcomes feed back to
              v
+-------------------------------------------+
|          Execution History (DB)           |
|  - Quality scores, latency, cost          |
|  - Used by GA for fitness computation     |
|  - Used by TS for alpha/beta updates      |
+-------------------------------------------+
```

1. The GA searches broadly across the configuration space, evaluating
   populations of candidates against historical data.
2. The fittest GA individuals are registered as new Thompson Sampling arms
   with a uniform prior (Beta(1,1)).
3. Thompson Sampling allocates real traffic to these arms, accumulating
   observations and narrowing their distributions.
4. Execution outcomes from Thompson Sampling flow back into the database,
   providing fresh data for the next GA generation.

This creates a virtuous cycle: the GA proposes, Thompson Sampling tests, and
the execution history informs both.

---

## Convergence Tracking

Each optimizer exposes convergence metrics through the REST API:

```
GET /api/ga/{optimizer}/convergence

{
  "optimizer": "model_routing",
  "generation": 23,
  "best_fitness": 0.847,
  "mean_fitness": 0.721,
  "fitness_variance": 0.018,
  "generations_since_improvement": 3,
  "converged": false,
  "population_diversity": 0.42
}
```

An optimizer is considered converged when `generations_since_improvement`
exceeds the configured threshold (default: 5) or when `population_diversity`
drops below 0.1 (indicating the population has collapsed to near-identical
individuals).

---

## Why GA Over Grid Search or Random Search

```
+--------------+--------+--------+---------+------------------------------+
| Method       | Dims=3 | Dims=6 | Dims=10 | Handles Dependencies?        |
+--------------+--------+--------+---------+------------------------------+
| Grid search  | 1K     | 1M     | 10B     | No (evaluates independently) |
|              | evals  | evals  | evals   |                              |
+--------------+--------+--------+---------+------------------------------+
| Random search| ~100   | ~500   | ~2000   | No (samples independently)   |
+--------------+--------+--------+---------+------------------------------+
| GA           | ~200   | ~600   | ~1000   | Yes (crossover preserves     |
|              |        |        |         | correlated gene combinations)|
+--------------+--------+--------+---------+------------------------------+
```

Grid search is exponential in the number of dimensions. With 6 genes and 10
values per gene, grid search requires 10^6 = 1,000,000 evaluations. A GA with
a population of 30 over 20 generations requires 600 evaluations to find
competitive solutions.

Random search handles dimensionality better than grid search but treats each
parameter independently. It cannot discover that `workers=6` works well
specifically when `consensus_type=weighted` and `timeout=120`. GAs discover
these correlations through crossover: successful gene combinations from one
parent propagate to offspring alongside successful genes from the other parent.

The tradeoff is that GAs are not guaranteed to find the global optimum. For the
orchestration framework's use case -- finding good-enough configurations from a
large, non-convex, multi-dimensional space with parameter dependencies -- GAs
provide the best balance of sample efficiency and solution quality.
