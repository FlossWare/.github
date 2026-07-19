# Deployment Guide

This guide covers deploying every component of the distributed LLM orchestration framework: the controller API, fleet workers, scraper pipelines, embedding workers, and backing datastores. Deployment is automated via Ansible.

---

## Automated Deployment (Ansible)

The preferred deployment method uses the Ansible playbooks in `ansible/`. The inventory organizes the fleet by CPU architecture (x86_64, ARM64/aarch64, ARMv7/armhf).

```bash
# Full fleet deployment (all 10 phases)
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/site.yml

# Deploy only workers
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/site.yml --tags workers

# Deploy only scrapers
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/site.yml --tags scraping

# Deploy only monitoring stack
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/site.yml --tags monitoring
```

The site playbook runs 10 phases in order: common baseline, NFS, PostgreSQL, Redis, REST API, monitoring, workers, GA tools, scraping, backups.

---

## Prerequisites

- **Ansible 2.14+** on the control machine (laptop-01)
- **Network access** to the orchestrator API (controller node, port 5000)
- **SSH access** to fleet worker nodes (key-based authentication, user `claude`)
- **PostgreSQL client** (optional, for direct schema inspection)
- **Node.js 18+** on all nodes that run workflows
- **Python 3.10+** on nodes that run scrapers or GA optimizers
- **Docker** on the controller node (for OrientDB)

---

## Controller Node (API)

The Flask-based orchestrator API runs on the controller node as a systemd service.

```bash
# Deploy the API
scp -r api/ claude@aio-01:/home/claude/orchestrator/api/
ssh claude@aio-01 "cd /home/claude/orchestrator && pip3 install -r requirements.txt"

# Install and enable the systemd service
scp scripts/orchestrator-api.service claude@aio-01:/etc/systemd/system/
ssh claude@aio-01 "sudo systemctl daemon-reload && sudo systemctl enable --now orchestrator-api"

# Verify
curl http://aio-01:5000/health
```

The API serves as the single entry point for all database access. Direct database connections from application code are prohibited; use the REST API exclusively.

---

## Fleet Worker Deployment

Workers join the fleet through SSH key exchange and circuit breaker registration. No configuration changes are needed on the orchestrator.

```bash
# 1. Distribute SSH key to the new worker
ssh-copy-id claude@<worker-host>

# 2. Verify connectivity
ssh claude@<worker-host> "hostname && python3 --version && node --version"

# 3. Register the worker's circuit breaker via API
curl -X POST http://aio-01:5000/fleet/register \
  -H "Content-Type: application/json" \
  -d '{"host": "<worker-host>", "capabilities": ["scrape", "chunk", "workflow"]}'

# 4. Confirm registration
curl http://aio-01:5000/fleet/status
```

Workers are automatically included in task distribution once registered. The circuit breaker tracks failures per worker and temporarily removes unhealthy nodes from the pool.

**Important:** Never run worker tasks on the controller node (aio-01). It is the orchestrator only.

---

## Scraper Deployment

Scrapers follow a deploy-and-launch pattern on individual workers.

```bash
# 1. Copy scraper files to the worker
scp scrapers/my_scraper.py claude@server-01:/home/claude/workers/scrapers/

# 2. Launch the scraper in the background
ssh claude@server-01 "cd /home/claude/workers/scrapers && nohup python3 my_scraper.py > my_scraper.log 2>&1 &"

# 3. Verify it is running and producing output
ssh claude@server-01 "tail -20 /home/claude/workers/scrapers/my_scraper.log"
```

All scrapers must subclass `BaseScraper`. The ingestion pipeline handles chunking, embedding, and graph insertion downstream.

---

## Store Worker Deployment

Store workers are REST-based queue consumers that persist processed data. They must use persistent filesystem paths.

```bash
# Correct: persistent path
WORK_DIR=/home/claude/workers/store

# WRONG: data is lost on reboot
WORK_DIR=/tmp/store   # NEVER do this
```

Deploy store workers the same way as scrapers: `scp` to the worker, `nohup` launch, verify via logs.

---

## Embedding Worker Deployment

Embedding workers run `sentence-transformers` models and require significant CPU or GPU resources. They must only run on dedicated compute nodes (laptop-01, laptop-02), never on fleet workers.

```bash
# On a dedicated compute node only
ssh claude@laptop-01 "cd /home/claude/workers/embed && nohup python3 embed_worker.py > embed.log 2>&1 &"
```

Embedding workers query the database directly for unprocessed chunks rather than consuming from a queue.

---

## Database Setup

### PostgreSQL with pgvector

PostgreSQL runs on the controller node (port 5433) with the `pgvector` extension for vector similarity search.

```bash
# Install pgvector extension
psql -h aio-01 -p 5433 -U sfloess -d learning -c "CREATE EXTENSION IF NOT EXISTS vector;"

# Run schema migrations
psql -h aio-01 -p 5433 -U sfloess -d learning -f schemas/learning.sql
psql -h aio-01 -p 5433 -U sfloess -d learning -f schemas/workflow.sql
psql -h aio-01 -p 5433 -U sfloess -d learning -f schemas/monitoring.sql
psql -h aio-01 -p 5433 -U sfloess -d learning -f schemas/costs.sql
```

### Redis

Redis runs in standalone mode on the controller node (port 6379, no authentication). It handles all queue operations.

```bash
# Verify Redis is running
redis-cli -h aio-01 ping
# Expected: PONG
```

### OrientDB

OrientDB runs as a Docker container on the controller node for graph-based knowledge storage.

```bash
ssh claude@aio-01 "docker start orientdb || docker run -d --name orientdb \
  -p 2424:2424 -p 2480:2480 \
  -v /home/claude/orientdb/databases:/orientdb/databases \
  orientechnologies/orientdb:latest"
```

---

## Health Verification

After deployment, verify all components:

```bash
# API health
curl http://aio-01:5000/health

# Fleet status
curl http://aio-01:5000/fleet/status

# Queue status
curl http://aio-01:5000/queue/status

# Database connectivity (via API)
curl http://aio-01:5000/db/health
```

A healthy system returns HTTP 200 on all endpoints with status fields indicating `"ok"` or `"healthy"`.
