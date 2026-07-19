# FlossWare

**Distributed AI orchestration. Multi-model consensus. Evolutionary optimization.**

FlossWare is an open-source engineering organization building infrastructure for the multi-AI era. Our flagship project is a distributed orchestration framework that coordinates 200+ AI models across a fleet of nodes, using adversarial verification and genetic algorithms to deliver results no single model can match.

---

## The Problem

Asking one AI model a question and trusting the answer is a single point of failure. Every model has blind spots, biases, and failure modes. The larger the model, the more confidently it can be wrong.

## Our Solution

Instead of trusting one model, **orchestrate many**. FlossWare coordinates 200+ models from 8+ providers (Anthropic, OpenAI, Google, Groq, Cerebras, DeepSeek, and others) across a distributed fleet. Models review each other's work through adversarial panels with zero overlap. Genetic algorithms evolve better configurations over time. The result: higher quality, lower cost, and continuous improvement without retraining a single model.

---

## Core Capabilities

### Multi-AI Consensus
Query 3-8 models simultaneously. An arbiter synthesizes the best answer with confidence scoring. Thompson Sampling learns which models excel at which tasks and routes accordingly.

### Adversarial Review Pipeline
The system's signature pattern. Code changes pass through four phases -- **Review**, **Meta-Review**, **Fix**, **Verify** -- where review and meta-review panels share **zero models** to prevent self-confirmation bias. Each phase uses a different arbiter. External reviewers (Grok, ChatGPT, Notebook LLM) provide independent adversarial critique outside the orchestrator's ecosystem.

### Knowledge Pipeline
88+ scrapers collect documents across 12 domains. A 4-stage async pipeline (store, chunk, embed, graph) converts raw content into searchable vector embeddings and graph-linked knowledge. 315,000+ documents, 705,000+ embeddings, 0.4ms vector search.

### Genetic Algorithm Evolution
Seven GA optimizers evolve model routing, team composition, RAG parameters, prompt templates, and workflow configurations using real execution history as fitness data. Best strategies feed into Thompson Sampling for real-time exploitation.

### Distributed Fleet
9 nodes (1 controller + 8 workers) spanning three CPU architectures (x86_64, ARM64, ARMv7) connected via SSH. Deployed and managed via Ansible with 11 roles. All 200+ models accessed via free-tier APIs. Circuit breakers, provider fallback, and rate limiting ensure reliability.

---

## Architecture at a Glance

```
┌─────────────────────────────┐
│     Clients / Agents        │
└──────────────┬──────────────┘
               │ REST API
┌──────────────▼──────────────┐
│   Orchestrator (Flask API)  │
│  Routing ─ Consensus ─ GA   │
└───────┬─────────────┬───────┘
        │ SSH         │ API
┌───────▼───────┐ ┌───▼────────┐
│ Worker Fleet  │ │ 200+ LLMs  │
│ (8 nodes)     │ │ (8+ provs) │
└───────────────┘ └────────────┘
        │
┌───────▼─────────────────────┐
│ PostgreSQL ─ Redis ─ OrientDB│
└─────────────────────────────┘
```

For the complete architecture, see **[ARCHITECTURE.md](../ARCHITECTURE.md)**.

---

## Engineering Philosophy

**Orchestration, not training.** The models are pre-trained and used as-is via API. All improvements come from smarter routing, better team composition, and evolved configurations.

**Independence over agreement.** Review panels share zero models with meta-review panels. External AI reviewers operate outside the orchestrator entirely. Quality comes from independent perspectives, not consensus theater.

**Evolution over configuration.** Don't hand-tune parameters. Let genetic algorithms search the configuration space using real execution history as fitness data.

**Interfaces over implementations.** Every component communicates through REST APIs. The orchestrator doesn't know if a worker is a server or a Raspberry Pi. Any component can be replaced without touching the others.

**Provider independence.** No vendor lock-in. If a provider goes down or raises prices, the system routes to alternatives automatically. The models will change; the orchestration endures.

For the full design philosophy, see **[docs/philosophy.md](../docs/philosophy.md)**.

---

## Repository Map

### AI Infrastructure

| Repository | Description |
|------------|-------------|
| **[consensus-ai](https://github.com/FlossWare/consensus-ai)** | Multi-AI orchestration library with 5 consensus strategies |
| **[knowledge-ai](https://github.com/FlossWare/knowledge-ai)** | Universal knowledge ingestion from any documentation format |
| **[skills-ai](https://github.com/FlossWare/skills-ai)** | Executable workflows for the AI ecosystem |
| **[vectordb-ai](https://github.com/FlossWare/vectordb-ai)** | Universal vector database adapter -- 9 backends, no vendor lock-in |
| **[semantic-search-ai](https://github.com/FlossWare/semantic-search-ai)** | Hybrid search, reranking, filtering for AI applications |
| **[netbeans-plugins](https://github.com/FlossWare/netbeans-plugins)** | NetBeans IDE plugins for Claude, Gemini, and ChatGPT |

### Java Infrastructure

| Repository | Description |
|------------|-------------|
| **[commons-java](https://github.com/FlossWare/commons-java)** | Shared utilities for SOAP clients, string operations, file handling |
| **[platform-java](https://github.com/FlossWare/platform-java)** | Multi-application isolation -- isolated classloaders, thread pools, security |
| **[curses-java](https://github.com/FlossWare/curses-java)** | Terminal UI library with 29 AWT-like widgets and ncurses backend |
| **[collections-java](https://github.com/FlossWare/collections-java)** | Collections backed by files, networking, and other storage |
| **[classloader-java](https://github.com/FlossWare/classloader-java)** | Universal ClassLoader supporting 30+ protocols |
| **[remote-java](https://github.com/FlossWare/remote-java)** | RPC framework with multi-format serialization (JSON/XML/YAML/MessagePack) |
| **[container-java](https://github.com/FlossWare/container-java)** | Universal container/orchestration abstraction (Kubernetes, Docker, Hazelcast) |
| **[cloudstorage-java](https://github.com/FlossWare/cloudstorage-java)** | Universal cloud storage abstraction (S3, Azure Blob, GCS, Drive, Dropbox) |
| **[filetransfer-java](https://github.com/FlossWare/filetransfer-java)** | Universal file transfer abstraction (SFTP, WebDAV, SMB/CIFS, FTP) |
| **[messaging-java](https://github.com/FlossWare/messaging-java)** | Universal messaging/cache abstraction (Kafka, RabbitMQ, Redis) |
| **[vcs-java](https://github.com/FlossWare/vcs-java)** | Universal version control abstraction (Git) |
| **[encrypt-java](https://github.com/FlossWare/encrypt-java)** | AES-256-GCM encryption library |
| **[eventbus-java](https://github.com/FlossWare/eventbus-java)** | Event bus and service registry for inter-application communication |
| **[threadpool-java](https://github.com/FlossWare/threadpool-java)** | Managed thread pools with monitoring and graceful shutdown |
| **[resource-monitor-java](https://github.com/FlossWare/resource-monitor-java)** | Resource usage tracking and quota enforcement |
| **[fs-watcher-java](https://github.com/FlossWare/fs-watcher-java)** | Filesystem watcher with debouncing |
| **[nexus-java](https://github.com/FlossWare/nexus-java)** | Nexus Repository Manager CLI/GUI with search, filtering, analytics |
| **[diskwipe-java](https://github.com/FlossWare/diskwipe-java)** | Secure disk space wiping with zero-fill operations |
| **[build-tools](https://github.com/FlossWare/build-tools)** | Automated code quality and refactoring tools |

### Systems and DevOps

| Repository | Description |
|------------|-------------|
| **[PxeOS](https://github.com/FlossWare/PxeOS)** | Cross-OS PXE boot provisioning (Linux, BSD, Windows) |
| **[VirtOS](https://github.com/FlossWare/VirtOS)** | Minimal virtualization OS based on Tiny Core Linux |
| **[VirtOS-Examples](https://github.com/FlossWare/VirtOS-Examples)** | Ready-to-deploy templates for VirtOS microservices |
| **[cobbler](https://github.com/FlossWare/cobbler)** | Cobbler templates for RHEL/Fedora, Debian/Ubuntu, FreeBSD provisioning |
| **[notion2config](https://github.com/FlossWare/notion2config)** | Generate system configs (dnsmasq, Ansible, nginx) from Notion databases |
| **[de-converter](https://github.com/FlossWare/de-converter)** | Convert desktop environment configs to lightweight window managers |

### Applications

| Repository | Description |
|------------|-------------|
| **[hotspot-android](https://github.com/FlossWare/hotspot-android)** | Free Android hotspot app for internet outages |
| **[Samsung-Galaxy-J7](https://github.com/FlossWare/Samsung-Galaxy-J7)** | Transform Galaxy J7 into a mini Linux computer |
| **[civilization-simulator-java](https://github.com/FlossWare/civilization-simulator-java)** | Alternate history civilization simulator |
| **[curses-themes](https://github.com/FlossWare/curses-themes)** | Lightweight theme support for Python curses applications |

### Documentation

| Repository | Description |
|------------|-------------|
| **[.github](https://github.com/FlossWare/.github)** | Organization profile and architecture documentation |
| **[FlossWare](https://github.com/FlossWare/FlossWare)** | Generated documentation (Javadocs) |

---

## Documentation

| Document | Description |
|----------|-------------|
| **[ARCHITECTURE.md](../ARCHITECTURE.md)** | Complete system architecture (20+ pages) |
| **[docs/philosophy.md](../docs/philosophy.md)** | Design philosophy and engineering reasoning |
| **[docs/architecture/](../docs/architecture/)** | Orchestration, consensus, fleet, routing |
| **[docs/knowledge/](../docs/knowledge/)** | Scraping, chunking, embeddings, graph |
| **[docs/databases/](../docs/databases/)** | PostgreSQL, Redis, OrientDB |
| **[docs/learning/](../docs/learning/)** | Thompson Sampling, genetic algorithms |
| **[docs/operations/](../docs/operations/)** | Deployment, monitoring, scaling |
| **[docs/development/](../docs/development/)** | Getting started, contributing, coding standards |

---

## Getting Started

```bash
# Clone the orchestration framework
git clone https://github.com/FlossWare/consensus-ai.git

# Or explore the architecture documentation
git clone https://github.com/FlossWare/.github.git
```

See **[docs/development/getting_started.md](../docs/development/getting_started.md)** for the full setup guide.

---

## Contributing

FlossWare is open to contributions from developers, AI practitioners, and infrastructure engineers. See **[docs/development/contributing.md](../docs/development/contributing.md)** for guidelines.

---

## Core Principle

**The models will change. The orchestration endures.**

---

*Built with multi-AI consensus. Reviewed by adversarial panels. Evolved by genetic algorithms.*
