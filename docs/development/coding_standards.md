# Coding Standards

This document defines the coding conventions for the distributed LLM orchestration framework. All contributions must follow these standards.

---

## JavaScript

### Module System

Use ES Modules exclusively. All JavaScript files must use the `.mjs` extension.

```javascript
// Correct: ES Module syntax with .mjs extension
import { getWorkflowStorage } from './shared/workflow-storage-adapter.js';
import { readFile } from 'node:fs/promises';

export default async function myWorkflow({ phase, parallel, agent }) {
  // ...
}

export const meta = { name: 'my-workflow' };
```

### Async/Await

Use `async`/`await` over raw promise chains or callbacks.

```javascript
// Correct
async function fetchModels() {
  const response = await fetch('http://aio-01:5000/api/strategies');
  const data = await response.json();
  return data.strategies;
}

// Avoid
function fetchModels() {
  return fetch('http://aio-01:5000/api/strategies')
    .then(r => r.json())
    .then(data => data.strategies);
}
```

### Error Handling

Catch errors and log them. Do not fail silently, and do not let unhandled rejections crash the process.

```javascript
try {
  const result = await agent('Analyze this code');
  await db.storeWorkerResult({ workflow_execution_id: execId, result });
} catch (err) {
  console.error(`Worker failed: ${err.message}`);
  await db.storeWorkerResult({
    workflow_execution_id: execId,
    result: null,
    error: err.message
  });
}
```

---

## Python

### Type Hints

All public functions must include type hints for parameters and return values.

```python
from typing import Dict, List, Optional

def analyze_feedback_loops(window_days: int = 7) -> Dict:
    """Analyze feedback loops over the specified time window."""
    ...

def find_similar_documents(query: str, limit: int = 10) -> List[Dict]:
    """Return documents similar to the query text."""
    ...
```

### Docstrings

Modules and public functions must have docstrings. Use imperative mood for the first line.

```python
"""Genetic algorithm optimizer for model routing weights.

Evolves routing parameters using tournament selection, uniform crossover,
and Gaussian mutation. Fitness is measured by downstream task success rate.
"""

def evaluate_chromosome(chromosome: Dict, trials: int = 50) -> float:
    """Evaluate a chromosome's fitness over the specified number of trials."""
    ...
```

### Exception Handling

Use `try`/`except` with specific exception types. Do not catch bare `Exception` unless re-raising or logging.

```python
# Correct
try:
    response = requests.post(url, json=payload, timeout=10)
    response.raise_for_status()
except requests.ConnectionError:
    logger.error(f"Cannot reach {url}")
    return None
except requests.Timeout:
    logger.warning(f"Request to {url} timed out")
    return None

# Avoid
try:
    response = requests.post(url, json=payload)
except:
    pass
```

---

## Database Access

### Use the REST API, Never Direct Connections

All database access must go through the orchestrator's REST API (port 5000). Direct connections to PostgreSQL (port 5433), Redis, or OrientDB from application code are prohibited.

```javascript
// Correct: REST API via shared adapter
const { getDB } = require('./shared/postgres-adapter.js');
const db = getDB();
const rows = await db.query('SELECT * FROM learning.experiences');

// Wrong: direct database connection
const { Pool } = require('pg');
const pool = new Pool({ host: 'aio-01', port: 5433 });  // NEVER do this
```

### Use Shared Adapters

Three adapters encapsulate common database patterns:

| Adapter                          | Purpose                                      |
|----------------------------------|----------------------------------------------|
| `shared/postgres-adapter.js`     | General database queries via REST API        |
| `shared/workflow-storage-adapter.js` | Workflow execution tracking              |
| `shared/feedback-loop-adapter.cjs`   | Feedback loop health checks              |

### Track All Workflows

Every multi-model workflow execution must be recorded via the workflow storage adapter. This data feeds the learning system and enables post-hoc analysis.

```javascript
const { getWorkflowStorage } = require('./shared/workflow-storage-adapter.js');
const db = getWorkflowStorage();

const execId = await db.storeExecution({
  workflow_id: 'wf-' + Date.now(),
  workflow_name: 'code-review',
  task_description: 'Review authentication module',
  total_workers: 6,
  total_duration_ms: 32000,
  outcome: 'success'
});

// Store each worker's result
await db.storeWorkerResult({
  workflow_execution_id: execId,
  worker_id: 'review-1',
  model: 'opus',
  result: 'Found 3 issues...'
});
```

---

## Workflow Patterns

### Zero Overlap Between Review and Meta-Review

When a workflow includes both a review phase and a meta-review phase, the two model panels must share zero models. This prevents self-confirmation bias.

```javascript
// Correct: completely separate panels
const REVIEW_MODELS = ['opus', 'sonnet', 'deepseek-chat', 'qwen3-coder'];
const META_REVIEW_MODELS = ['fable', 'hermes-405b', 'nemotron-ultra', 'qwen3-next'];

// Wrong: opus and sonnet appear in both panels
const REVIEW_MODELS = ['opus', 'sonnet', 'haiku'];
const META_REVIEW_MODELS = ['opus', 'sonnet', 'fable'];
```

### Different Arbiter Per Phase

Each workflow phase must use a different arbiter model to synthesize results. Reusing the same arbiter across phases undermines the independence of each evaluation.

```javascript
const REVIEW_ARBITER = 'opus';
const META_REVIEW_ARBITER = 'sonnet';
const SOLVE_ARBITER = 'fable';
const VERIFY_ARBITER = 'haiku';
```

### Meta-Reviewers Must Be Strong Models

Meta-review models must be capable enough to adversarially challenge the review panel's findings. Using weak models for meta-review defeats its purpose.

```javascript
// Wrong: haiku cannot effectively challenge opus/sonnet findings
const META_REVIEW = ['haiku', 'haiku', 'haiku'];

// Correct: strong models that can push back on review findings
const META_REVIEW = ['fable', 'hermes-405b', 'nemotron-ultra-550b', 'qwen3-next-80b'];
```

---

## General Conventions

### No Hardcoded Model Names

Do not hardcode model names in task execution calls. Let the routing layer select the appropriate model based on task type and current performance data.

```javascript
// Correct: let routing decide
const result = await agent('Analyze this code for security issues');

// Avoid: bypasses routing intelligence
const result = await agent('Analyze this code', { model: 'opus' });
```

Model names are acceptable in panel definitions (review, meta-review) where specific composition is intentional.

### Rate-Limit External API Calls

Respect provider rate limits. Use the built-in retry logic with exponential backoff rather than flooding endpoints.

### Use Persistent Paths

Never use `/tmp` for any data that must survive a reboot. Worker data, scraper output, and queue state must reside on persistent paths.

```bash
# Correct
/home/claude/workers/scrapers/mydocs/

# Wrong: lost on reboot
/tmp/mydocs/
```

### Idempotent Operations

Design operations to be safely re-runnable. Scrapers should skip already-processed URLs. Pipeline stages should detect and skip duplicate documents. Database writes should use upserts where appropriate.

### Store Execution Data

All execution data must be persisted for the learning system. Every model call, workflow execution, and pipeline run should produce records that can be analyzed later. The system improves its routing and task decomposition based on this historical data.
