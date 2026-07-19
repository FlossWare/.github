# Redis Architecture

Redis serves three roles in the orchestration framework: pipeline queue
management, API response caching, and per-provider rate limiting. It runs in
standalone mode on the controller node (aio-01) and is accessed exclusively
through the REST API gateway.

**Version:** Redis 8.0
**Location:** Controller node (aio-01), port 6379
**Authentication:** None (bound to private network interface)
**Persistence:** RDB snapshots every 60 seconds if 100+ keys changed

---

## Pipeline Queues

The document ingestion pipeline has four stages. Each stage has a dedicated
Redis sorted set that functions as a priority queue.

```
+-------+     +-------+     +-------+     +-------+
| store | --> | chunk | --> | embed | --> | graph |
+-------+     +-------+     +-------+     +-------+
   ZSET          ZSET          ZSET          ZSET

  pipeline:store:pending    pipeline:chunk:pending
  pipeline:store:active     pipeline:chunk:active
  pipeline:embed:pending    pipeline:graph:pending
  pipeline:embed:active     pipeline:graph:active
```

Each stage maintains two sorted sets: `pending` (items awaiting processing) and
`active` (items currently claimed by a worker). A dead-letter queue
(`pipeline:<stage>:dlq`) holds items that exceeded the retry limit.

### Priority Scoring

Items are scored so that priority always dominates over arrival time, with FIFO
ordering within the same priority band:

```
score = (10 - priority) * 1e13 + timestamp_ms
```

- `priority` ranges from 1 (highest) to 9 (lowest)
- `timestamp_ms` is Unix epoch in milliseconds
- The `1e13` multiplier ensures that a priority-1 item added at any time sorts
  before a priority-2 item, regardless of when the priority-2 item arrived

Example scores for two items:

```
Priority 1, added at T=1000:  9 * 1e13 + 1000 = 90000000001000
Priority 2, added at T=500:   8 * 1e13 + 500  = 80000000000500

ZPOPMIN returns: priority-2 item first? No -- higher score = lower priority
in ZPOPMIN. Items with LOWER scores are popped first.

Correction: ZPOPMIN pops the LOWEST score.
  Priority 1: (10-1) * 1e13 + 1000 = 90000000001000
  Priority 2: (10-2) * 1e13 + 500  = 80000000000500

The priority-2 item has score 8e13, which is LOWER, so ZPOPMIN would take it
first. That is wrong -- priority 1 should go first.

The actual formula inverts:
  score = priority * 1e13 + timestamp_ms

  Priority 1: 1 * 1e13 + 1000 = 10000000001000
  Priority 2: 2 * 1e13 + 500  = 20000000000500

Now ZPOPMIN correctly returns the priority-1 item first (lower score).
```

### Lua Scripts for Atomic Operations

Six Lua scripts execute atomically on the Redis server, preventing race
conditions between concurrent workers.

| Script    | Operation                                                  |
|-----------|------------------------------------------------------------|
| ADD       | Insert item into pending set with computed score           |
| FETCH     | ZPOPMIN from pending, add to active with ownership token   |
| COMPLETE  | Verify ownership, remove from active, chain to next stage |
| FAIL      | Increment retry count; route to DLQ if limit exceeded      |
| RECLAIM   | Move items from active back to pending if heartbeat stale  |
| HEARTBEAT | Update worker's last-seen timestamp for claimed items      |

The FETCH script illustrates the claim pattern:

```lua
-- FETCH: atomically claim the highest-priority item
-- KEYS[1] = pending set, KEYS[2] = active set
-- ARGV[1] = worker_id, ARGV[2] = current_timestamp

local item = redis.call('ZPOPMIN', KEYS[1])
if #item == 0 then return nil end

local payload = item[1]
local score = item[2]

-- Add to active set with worker ownership
local active_key = payload .. '::' .. ARGV[1]
redis.call('ZADD', KEYS[2], ARGV[2], active_key)

return {payload, score}
```

### Automatic Pipeline Chaining

When a worker completes a stage, the COMPLETE script automatically enqueues
the item into the next stage's pending set:

```
PIPELINE_NEXT = {
    store: "chunk",
    chunk: "embed",
    embed: "graph"
}
```

The `graph` stage has no successor. Completing it marks the item as fully
processed and removes it from all queue state.

### Idempotency and Deduplication

Each item carries an idempotency key (typically a content hash). The ADD script
checks a deduplication set before inserting:

```
pipeline:<stage>:seen    -- SET of processed idempotency keys
```

If the key exists in the seen set, the ADD script returns without inserting.
Keys expire from the seen set after 24 hours to allow reprocessing of updated
content.

### Worker Heartbeats

Workers send heartbeats every 30 seconds for each claimed item. The RECLAIM
script runs on a 90-second interval and moves any item in the active set whose
last heartbeat exceeds 90 seconds back to the pending set. This handles worker
crashes without manual intervention.

```
Worker crashes at T=100
Last heartbeat at T=90
RECLAIM runs at T=180 (or next interval)
  -> active item with heartbeat < (180 - 90) = 90? Yes
  -> Move back to pending with original score
```

---

## API Response Caching

Consensus results from multi-model calls are cached to avoid redundant
computation when the same (or sufficiently similar) query appears within a
short window.

```
cache:consensus:<hash>   -- STRING with 15-minute TTL
cache:model:<hash>       -- STRING with 15-minute TTL
```

The cache key is a SHA-256 hash of the normalized prompt text. Cache hits
bypass the entire multi-model fan-out, returning the previously computed
consensus directly.

Cache invalidation is purely TTL-based. No manual invalidation is needed
because the cached values are advisory (consensus on a prompt), not
authoritative (database records).

---

## Rate Limiting

Per-provider rate limit counters track API usage and trigger automatic
rerouting when a provider's limits are approached.

```
ratelimit:<provider>:rpm     -- STRING with 60-second TTL (requests per minute)
ratelimit:<provider>:tpm     -- STRING with 60-second TTL (tokens per minute)
ratelimit:<provider>:daily   -- STRING with 24-hour TTL (daily quota)
```

The model router checks these counters before dispatching a request. If a
counter exceeds the configured threshold (typically 80% of the provider's
stated limit), the router falls back to alternative providers in priority
order.

```
Request for "opus" model
  -> Check ratelimit:anthropic:rpm = 47 (limit: 60, threshold: 48)
  -> Under threshold, proceed with Anthropic
  -> INCR ratelimit:anthropic:rpm

Request for "opus" model (later)
  -> Check ratelimit:anthropic:rpm = 49 (over threshold)
  -> Fallback to openrouter (same model, different provider)
  -> INCR ratelimit:openrouter:rpm
```

---

## Why Redis Over PostgreSQL for Queues

Both PostgreSQL and Redis can implement job queues. The choice of Redis for
this workload is driven by three factors:

```
+----------------------------+-----------+------------+
| Concern                    | Redis     | PostgreSQL |
+----------------------------+-----------+------------+
| Pop highest-priority item  | O(log n)  | O(n log n) |
|   (ZPOPMIN vs SELECT...    |  ZPOPMIN  |  FOR UPDATE|
|    FOR UPDATE SKIP LOCKED) |           |  SKIP LOCK |
+----------------------------+-----------+------------+
| Atomic claim + heartbeat   | Lua script| BEGIN ...  |
|                            | (single   |  COMMIT    |
|                            |  round-   | (multiple  |
|                            |  trip)    |  round-    |
|                            |           |  trips)    |
+----------------------------+-----------+------------+
| Data lifetime              | Ephemeral | Persistent |
|                            | (minutes  | (forever)  |
|                            |  to hours)|            |
+----------------------------+-----------+------------+
```

Queue items are inherently ephemeral. They exist for seconds to minutes before
being consumed. PostgreSQL's durability guarantees (WAL writes, fsync) add
latency that provides no value for data that will be deleted momentarily.

Redis sorted sets are purpose-built for this access pattern: insert with score,
pop the minimum, check membership. The equivalent PostgreSQL query
(`SELECT ... ORDER BY score LIMIT 1 FOR UPDATE SKIP LOCKED`) requires an index
scan, row lock acquisition, and transaction commit for each dequeue.

Persistent data (execution results, embeddings, metadata) goes to PostgreSQL.
Ephemeral coordination state (queues, heartbeats, rate counters) goes to Redis.
Each system handles the workload it was designed for.
