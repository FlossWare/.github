# Monitoring and Observability

This guide covers the monitoring infrastructure for the distributed LLM orchestration framework: fleet health, API provider tracking, execution logging, queue metrics, and automated feedback loop analysis.

---

## Fleet Health Monitoring

Each fleet worker is monitored via periodic health checks with circuit breaker protection.

```bash
# Check fleet health
curl http://aio-01:5000/fleet/status
```

The response includes per-worker status:

| Field              | Description                                           |
|--------------------|-------------------------------------------------------|
| `host`             | Worker hostname                                       |
| `status`           | `healthy`, `degraded`, or `offline`                   |
| `circuit_state`    | `closed` (normal), `open` (failing), `half-open` (testing) |
| `last_check`       | Timestamp of most recent health check                 |
| `failure_count`    | Consecutive failures since last success               |

Health check timeout is 2000ms. Workers that exceed the failure threshold are automatically removed from task distribution until they recover and the circuit breaker transitions back to `closed`.

---

## API Provider Health

The system tracks health per LLM API provider (OpenRouter, Anthropic, Google, Groq, Cerebras, DeepSeek, and others). Each provider transitions through three states:

- **healthy** -- normal operation, all requests routed
- **degraded** -- elevated error rate, reduced routing weight
- **disabled** -- temporarily removed from routing after sustained failures

Provider health is updated automatically based on response success rates. Disabled providers are re-tested periodically.

```bash
# View provider health
curl http://aio-01:5000/providers/health
```

---

## Execution Logging

All model executions are recorded in PostgreSQL for post-hoc analysis.

| Table                              | Purpose                                    |
|------------------------------------|--------------------------------------------|
| `monitoring.execution_summary`     | Aggregated execution stats per model/task  |
| `monitoring.execution_log`         | Individual execution records with timing   |

Key fields in `execution_log`:

- `model` -- which model was used
- `provider` -- which API provider served the request
- `task_type` -- classification of the task
- `latency_ms` -- end-to-end response time
- `token_count` -- tokens consumed (input + output)
- `success` -- whether the execution succeeded
- `error_message` -- failure reason if applicable

Query example via the REST API:

```bash
# Recent execution stats
curl "http://aio-01:5000/api/executions?limit=50&model=opus"
```

---

## Queue Monitoring

All queues are backed by Redis. Two endpoints expose queue state:

### Per-Queue and Aggregate Counts

```bash
curl http://aio-01:5000/queue/status
```

Returns the number of pending, processing, completed, and failed items for each queue (ingestion, chunking, graph insertion), plus aggregate totals.

### Prometheus-Compatible Metrics

```bash
curl http://aio-01:5000/queue/metrics
```

Returns metrics in Prometheus exposition format, suitable for scraping:

```
# HELP queue_pending Number of items waiting to be processed
# TYPE queue_pending gauge
queue_pending{queue="ingestion"} 42
queue_pending{queue="chunking"} 17
queue_pending{queue="graph"} 3
```

---

## Network Latency

The `monitoring.network_latency_matrix` table tracks round-trip latency between every pair of nodes in the fleet. This data informs task placement decisions -- latency-sensitive operations are routed to nearby nodes.

```bash
# View latency matrix
curl http://aio-01:5000/fleet/latency
```

---

## Model Diversity Monitoring

The `monitoring.diversity_alerts` table flags when model usage distribution becomes skewed. An alert fires when any single model accounts for more than 70% of executions within a rolling window.

Why this matters: over-reliance on one model creates a single point of failure and reduces the value of multi-model consensus.

```bash
# Check recent diversity alerts
curl http://aio-01:5000/monitoring/diversity-alerts
```

---

## Feedback Loop Monitoring

An automated 4-layer analysis runs every 6 hours to detect self-referential feedback loops. Each layer targets a specific anti-pattern:

| Layer | Detection Target                  | Threshold     |
|-------|-----------------------------------|---------------|
| 1     | Model dominance                   | >70% usage    |
| 2     | Evaluator-generator coupling      | >40% self-eval|
| 3     | Reward hacking                    | Quality up + diversity down |
| 4     | Concept collapse                  | Embedding similarity >0.90 |

The analysis is executed by `tools/feedback_loop_optimizer.py` and orchestrated by cron via `~/bin/monitor-feedback-loops.sh`.

```bash
# Run analysis manually
python3 tools/feedback_loop_optimizer.py --window 7 --output /tmp/report.json

# Check latest automated report
cat ~/.claude/reports/feedback-loops/latest.json
```

Exit codes: `0` = no critical risks, `1` = high-severity risks detected, `2` = critical risks requiring immediate attention.

The JavaScript adapter (`shared/feedback-loop-adapter.cjs`) provides programmatic access:

```javascript
const { isSystemHealthy, analyzeFeedbackLoops } = require('./shared/feedback-loop-adapter.cjs');

const healthy = await isSystemHealthy(7);  // 7-day window
if (!healthy) {
  const report = await analyzeFeedbackLoops({ windowDays: 7 });
  console.log(report.risks);
}
```

---

## Cost Tracking

The `costs.*` schema tracks API spending per provider, model, and time period.

```bash
# View cost summary
curl http://aio-01:5000/costs/summary?period=7d
```

This returns a breakdown of token usage and estimated cost by provider. Since the system primarily uses free-tier models via OpenRouter, costs are typically dominated by Anthropic API usage.

---

## Resource Usage

The `monitoring.resource_usage` table tracks per-worker resource utilization (CPU, memory, disk) over time. This data supports capacity planning and identifies workers that are over- or under-utilized.

```bash
# View current resource usage
curl http://aio-01:5000/fleet/resources
```

---

## Alerting

Alerts are stored in PostgreSQL (`monitoring.diversity_alerts` and feedback loop reports) rather than pushed to external systems. Operators should periodically check:

1. Fleet status for offline workers
2. Queue status for growing backlogs
3. Diversity alerts for model skew
4. Feedback loop reports for systemic risks
5. Cost summaries for unexpected spending
