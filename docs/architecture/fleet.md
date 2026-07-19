# Distributed Fleet

The fleet is a heterogeneous cluster of 9 machines that execute tasks in parallel. One node (aio-01) serves as the controller, running the orchestration API and dispatching work. The remaining 8 nodes are workers that execute model inference calls, run workflow scripts, and return results.

---

## Fleet Topology

```
                           FLEET TOPOLOGY

                        +------------------+
                        |     aio-01       |
                        |   CONTROLLER     |
                        |   Flask API      |
                        |   PostgreSQL     |
                        |   Redis Sentinel |
                        |   OrientDB      |
                        +--------+---------+
                                 |
                 SSH (BatchMode=yes, port 22)
                                 |
        +----------+-------------+-------------+-----------+
        |          |             |             |           |
  +-----+----+ +--+-------+ +--+-------+ +---+------+ +--+-------+
  | server-01| | server-02| | server-03| | laptop-01| | laptop-02|
  |  8C/16GB | |  8C/32GB | |  8C/32GB | |  4C/31GB | |  4C/31GB |
  |  x86_64  | |  x86_64  | |  x86_64  | |  x86_64  | |  x86_64  |
  |  Worker  | |  Worker  | |  Worker  | | Worker+  | |  Worker  |
  |          | |          | |          | | Embeds   | |          |
  +----------+ +----------+ +----------+ +----------+ +----------+

  +----------+ +----------+ +----------+ +----------+
  | desktop-ap| | server-ap| |  pi-01   | |  pi-02   |
  |   1GB     | |   1GB    | |  1GB     | |  1GB     |
  |  armv7l   | |  armv7l  | | aarch64  | | aarch64  |
  |  Worker   | |  Worker  | |  Worker  | |  Worker  |
  |  (light)  | |  (light) | |  (light) | |  (light) |
  +----------+ +----------+ +----------+ +----------+
```

---

## Node Roles

The fleet enforces strict role separation between the controller and workers.

### Controller (aio-01)

The controller runs the orchestration API and all supporting infrastructure. It never executes worker tasks.

**Runs:**
- Flask API (port 5000) -- the single integration surface
- PostgreSQL (port 5433) -- transactional and vector storage
- Redis Sentinel -- caching, queuing, rate limiting
- OrientDB (Docker, port 2424) -- graph database
- Cron jobs -- backups, monitoring, feedback loop analysis

**Never runs:**
- Model inference calls (API or local)
- Workflow script execution
- Embedding generation
- Scraping workers

The rationale for this separation: the controller must remain responsive to incoming API requests. If it is simultaneously running CPU-intensive or memory-intensive worker tasks, API latency degrades and health checks fail.

### Workers (8 nodes)

Workers execute tasks dispatched by the controller via SSH. They make outbound API calls to model providers, run workflow scripts, and return results to the controller.

**Runs:**
- Model inference via provider APIs (OpenRouter, Anthropic, Google, Groq, etc.)
- Workflow script execution (Node.js)
- Scraping tasks
- GA fitness evaluations

**Never runs:**
- Database operations (all database access goes through the controller's REST API)
- Embedding generation (restricted to laptop-01 and laptop-02 due to sentence-transformers memory requirements)
- Queue management (Redis is accessed only by the controller)

### Worker Tiers

Not all workers are equal. The fleet is divided into tiers based on hardware capability.

| Tier | Nodes | CPU | RAM | Arch | Best For |
|------|-------|-----|-----|------|----------|
| Heavy | server-01, server-02, server-03 | 8 cores | 16-32 GB | x86_64 | Parallel API calls, large context, multi-step workflows |
| Medium | laptop-01, laptop-02 | 4 cores | 31 GB | x86_64 | Embedding generation (exclusive), general tasks |
| Light | pi-01, pi-02 | 4 cores | 1 GB | aarch64 (ARM64) | Simple API calls, classification, fast-path consensus |
| Light | desktop-ap, server-ap | 4 cores | 1 GB | armv7l (armhf) | Simple API calls, classification, fast-path consensus |

The task dispatcher considers worker tier when assigning tasks. Code review (which requires holding multiple file contents in memory for context) goes to heavy-tier workers. Simple classification tasks go to light-tier workers. Embedding generation goes exclusively to medium-tier workers (laptop-01 and laptop-02) because sentence-transformers requires 2-4 GB of RAM for model loading.

---

## SSH-Based Distribution

All task distribution uses SSH with the following configuration.

```
Host *
    User claude
    BatchMode yes
    StrictHostKeyChecking accept-new
    ConnectTimeout 5
    ServerAliveInterval 15
    ServerAliveCountMax 3
```

**BatchMode=yes.** Disables interactive password prompts. If key-based authentication fails, the connection fails immediately rather than hanging. This is critical for automated dispatch -- a hung SSH connection would block the worker slot.

**StrictHostKeyChecking=accept-new.** Automatically accepts host keys on first connection but rejects changed keys (preventing MITM attacks on known hosts). This simplifies new worker onboarding while maintaining security for established nodes.

**ConnectTimeout=5.** If a worker does not respond within 5 seconds, the connection attempt is abandoned and the task is rerouted to another worker. Fleet workers on the local network typically connect in under 100ms; a 5-second timeout indicates the node is down or unreachable.

**ServerAliveInterval=15 / ServerAliveCountMax=3.** The SSH client sends a keepalive every 15 seconds and drops the connection after 3 missed responses (45 seconds of silence). This detects mid-task worker failures faster than waiting for the command to time out.

### Why SSH Instead of Kubernetes

The fleet is heterogeneous: x86_64 servers with 8 cores and 16-32 GB RAM alongside ARM64 Raspberry Pis and ARMv7 router-based workers with 1 GB RAM. Kubernetes assumes a relatively homogeneous cluster and adds significant overhead for container scheduling, pod networking, service mesh, and control plane operation.

SSH-based distribution has several advantages for this environment:

- **Zero infrastructure overhead.** No container runtime, no kubelet, no etcd. Workers need only an SSH daemon and Node.js.
- **Heterogeneous hardware.** Task dispatch can account for each node's actual capabilities. Kubernetes scheduling can do this too, but the configuration complexity is disproportionate for a 9-node fleet.
- **Simplicity.** A failed task dispatch is a failed SSH command with an exit code and stderr. Debugging does not require knowledge of pod lifecycle, container networking, or CRI internals.
- **Low resource consumption on constrained nodes.** The Raspberry Pis have 1 GB of RAM. A kubelet alone can consume 200-400 MB. SSH daemon uses approximately 2 MB.

The tradeoff: SSH-based distribution lacks automatic rescheduling, resource quotas, and service discovery that Kubernetes provides. These are implemented in the orchestration layer (circuit breakers serve as health-aware rescheduling, the node registry serves as service discovery, and task dispatch respects worker tier constraints).

---

## Health Monitoring

Each worker is monitored by the controller via periodic health checks.

**Health check interval.** Every 30 seconds.

**Health check method.** SSH to the worker and execute a lightweight probe command that verifies:
1. SSH connectivity (connection established within 5 seconds)
2. Disk space (>500 MB free on /home)
3. Memory availability (>100 MB free)
4. Node.js runtime accessible
5. Outbound HTTPS connectivity (can reach api.openrouter.ai)

**Health states:**

| State | Meaning | Action |
|-------|---------|--------|
| Healthy | All checks pass | Eligible for task dispatch |
| Degraded | Non-critical check fails (e.g., low disk) | Eligible but deprioritized |
| Unhealthy | Critical check fails (SSH timeout, no memory) | Excluded from dispatch |
| Unknown | No health data (new node or stale data) | Treated as Unhealthy |

Health state transitions are logged to `monitoring.execution_summary` and trigger alerts when a previously healthy node becomes unhealthy.

### Per-Node Circuit Breakers

Independent of provider-level circuit breakers, each worker node has its own circuit breaker.

- **Threshold:** 3 consecutive task failures (SSH timeout, non-zero exit, or output parsing failure)
- **Cooldown:** 60 seconds
- **Half-open:** 1 probe task dispatched to test recovery
- **Maximum cooldown:** 5 minutes (exponential backoff)

A worker circuit breaker trip does not affect other workers. The orchestrator simply stops dispatching to that node until it recovers.

---

## Deployment Automation (Ansible)

Fleet deployment is automated via Ansible with 11 roles and a 10-phase site playbook.

**Inventory structure:**

The Ansible inventory organizes the fleet by CPU architecture, which determines package availability and binary compatibility:

```yaml
workers:
  children:
    x86_workers:       # server-01, server-02, server-03
    arm64_workers:     # pi-01, pi-02 (aarch64)
    armv7_workers:     # desktop-ap, server-ap (armhf)
```

**Roles:**

| Role | Target | Purpose |
|------|--------|---------|
| `common` | All nodes | Baseline packages, SSH config, user setup |
| `nfs` | Controller (server), workers (client) | Shared filesystem |
| `postgresql` | Controller | PostgreSQL + pgvector, schema migrations |
| `redis` | Controller | Redis queue and cache |
| `unified_api` | Controller | Flask API + Gunicorn |
| `monitoring` | All nodes, controller, Grafana host | node_exporter, Prometheus, Alertmanager, Grafana |
| `worker` | Workers | Worker runtime, fleet executor |
| `ga_tools` | Controller | Genetic algorithm optimizers |
| `scraping` | Workers | Scraper deployment and configuration |
| `orientdb` | Controller | OrientDB Docker container |
| `backup` | Controller | Automated database backups |

**Deployment:**

```bash
# Full fleet deployment (all 10 phases)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml

# Targeted deployment by tag
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags workers
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags scraping
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring
```

---

## Scaling

Adding a new worker to the fleet requires two steps.

**Step 1: SSH key deployment.**

```bash
# On the new worker
sudo useradd -m -s /bin/bash claude
sudo mkdir -p /home/claude/.ssh
# Copy the controller's public key to /home/claude/.ssh/authorized_keys
sudo chown -R claude:claude /home/claude/.ssh
sudo chmod 700 /home/claude/.ssh
sudo chmod 600 /home/claude/.ssh/authorized_keys
```

**Step 2: Register with the controller.**

```bash
# POST to the fleet API
curl -X POST http://aio-01:5000/api/fleet/register \
  -H "Content-Type: application/json" \
  -d '{
    "hostname": "new-worker-01",
    "address": "192.168.1.50",
    "cores": 4,
    "ram_gb": 8,
    "tier": "medium",
    "capabilities": ["inference", "workflows"]
  }'
```

The controller immediately begins health-checking the new node. Once it passes, it enters the dispatch pool. Thompson Sampling initializes the new node with neutral priors, so it receives exploratory task assignments until the system learns its performance characteristics.

No restart of the controller or any existing worker is required.

### Removing a Worker

Workers can be removed gracefully or ungracefully.

**Graceful:** Call `DELETE /api/fleet/{hostname}`. The controller stops dispatching new tasks, waits for in-flight tasks to complete (up to 60 seconds), then removes the node from the registry.

**Ungraceful:** The node goes offline. The per-node circuit breaker trips after 3 failed dispatch attempts (approximately 90 seconds). In-flight tasks on that node time out and are retried on other workers.

---

## Graceful Degradation

The fleet is designed to continue operating with reduced capacity rather than failing entirely.

| Workers Available | Behavior |
|-------------------|----------|
| 8 (full fleet) | Normal operation, maximum parallelism |
| 5-7 | Normal operation, slightly reduced throughput |
| 3-4 | Reduced parallelism, light-tier tasks may queue |
| 2 | Minimum viable fleet, consensus limited to 2-3 models |
| 1 | Single-worker mode, no consensus (direct model call) |
| 0 | Controller executes locally as last resort |

When fewer than 3 workers are available, the system reduces consensus worker count to match availability and emits a warning. When no workers are available, the controller can execute tasks locally as a last resort (it has Node.js and outbound API access), but this is discouraged because it violates the role separation principle and can degrade API responsiveness.

The fleet abort threshold is configurable. The default behavior is: if more than half of dispatched workers fail for a single task, abort the consensus and retry the entire task once. If the retry also sees majority worker failure, return an error to the caller.

---

## Operational Constraints

Several constraints are enforced across the fleet to maintain system stability.

**Never run embeddings on workers.** The sentence-transformers library loads a 400MB model into memory on first invocation. On 1 GB workers (Raspberry Pis, desktop-ap, server-ap), this causes OOM kills. Embedding generation is restricted to laptop-01 and laptop-02 (31 GB RAM each).

**Never run worker tasks on the controller.** The controller must remain responsive for API requests, health checks, and cron jobs. Saturating its CPU or memory with worker tasks degrades the entire fleet.

**Worker storage is ephemeral.** Workers use `/home/claude/workers/` for temporary task files. This directory is cleaned after each task completes. Never use `/tmp` (lost on reboot) and never assume persistence across tasks.

**All database access goes through the REST API.** Workers never connect directly to PostgreSQL, Redis, or OrientDB. This centralizes connection management, enforces schema validation, and simplifies firewall rules (workers need only outbound SSH and HTTPS).
