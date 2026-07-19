# FlossWare

**Distributed AI orchestration. Multi-model consensus. Evolutionary optimization.**

FlossWare is an open-source platform for orchestrating independent AI systems at scale. Rather than relying on a single language model, FlossWare coordinates hundreds of AI models, distributed workers, and knowledge services through consensus, adversarial verification, and evolutionary optimization.

---

## The Problem

Asking one AI model a question and trusting the answer is a single point of failure. Every model has blind spots, biases, and failure modes. The larger the model, the more confidently it can be wrong.

## Our Solution

Instead of trusting one model, **orchestrate many**. FlossWare coordinates 200+ models from 8+ providers (Anthropic, OpenAI, Google, Groq, Cerebras, DeepSeek, and others) across a distributed fleet. Models review each other's work through adversarial panels with zero overlap. Genetic algorithms evolve better configurations over time. The result is designed to produce more reliable results than traditional single-model workflows through independent verification and multi-model consensus.

---

## Why FlossWare?

Most AI frameworks focus on prompting a single model.

FlossWare focuses on orchestrating many independent models.

The value is not any individual model, but the system that coordinates them through consensus, adversarial review, knowledge retrieval, and evolutionary optimization. The architecture treats AI models as interchangeable components while continuously improving how they are used.

---

## Engineering Philosophy

**Orchestration, not training.** The models are pre-trained and used as-is via API. All improvements come from smarter routing, better team composition, and evolved configurations.

**Independence over agreement.** Review panels share zero models with meta-review panels. External AI reviewers operate outside the orchestrator entirely. Quality comes from independent perspectives, not consensus theater.

**Evolution over configuration.** Don't hand-tune parameters. Let genetic algorithms search the configuration space using real execution history as fitness data.

**Interfaces over implementations.** Every component communicates through REST APIs. The orchestrator doesn't know if a worker is a server or a Raspberry Pi. Any component can be replaced without touching the others.

**Provider independence.** No vendor lock-in. If a provider goes down or raises prices, the system routes to alternatives automatically. The models will change; the orchestration endures.

For the full design philosophy, see **[Design Philosophy](https://github.com/FlossWare/.github/blob/main/docs/philosophy.md)**.

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
└──────┬──────────┬───────────┘
       │ SSH      │ API
┌──────▼──────┐ ┌─▼───────────┐
│ Worker Fleet│ │ 200+ LLMs   │
│ (8 nodes)   │ │ (8+ provs)  │
└──────┬──────┘ └─────────────┘
       │
┌──────▼──────────────────────┐
│   Knowledge Platform        │
│ PostgreSQL ─ Redis ─ OrientDB│
│ Vectors ─ Queues ─ Graph    │
└──────┬──────────────────────┘
       │
┌──────▼──────────────────────┐
│   Learning Engine           │
│ Thompson Sampling ─ GA ─    │
│ Feedback Loop Optimizer     │
└─────────────────────────────┘
```

For the complete architecture, see **[ARCHITECTURE.md](https://github.com/FlossWare/.github/blob/main/ARCHITECTURE.md)**.

---

## Repository Map

FlossWare consists of 37 repositories organized into AI infrastructure, Java infrastructure, systems software, applications, and documentation.

### Flagship Projects

| Repository | Description |
|------------|-------------|
| **[consensus-ai](https://github.com/FlossWare/consensus-ai)** | Multi-AI orchestration library with 5 consensus strategies |
| **[knowledge-ai](https://github.com/FlossWare/knowledge-ai)** | Universal knowledge ingestion from any documentation format |
| **[skills-ai](https://github.com/FlossWare/skills-ai)** | Executable workflows for the AI ecosystem |

### Core AI Libraries

| Repository | Description |
|------------|-------------|
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
| **[ARCHITECTURE.md](https://github.com/FlossWare/.github/blob/main/ARCHITECTURE.md)** | Complete system architecture (20+ pages) |
| **[Design Philosophy](https://github.com/FlossWare/.github/blob/main/docs/philosophy.md)** | Design philosophy and engineering reasoning |
| **[Orchestration](https://github.com/FlossWare/.github/tree/main/docs/architecture)** | Orchestration, consensus, fleet, routing |
| **[Knowledge Pipeline](https://github.com/FlossWare/.github/tree/main/docs/knowledge)** | Scraping, chunking, embeddings, graph |
| **[Databases](https://github.com/FlossWare/.github/tree/main/docs/databases)** | PostgreSQL, Redis, OrientDB |
| **[Learning](https://github.com/FlossWare/.github/tree/main/docs/learning)** | Thompson Sampling, genetic algorithms |
| **[Operations](https://github.com/FlossWare/.github/tree/main/docs/operations)** | Deployment, monitoring, scaling |
| **[Development](https://github.com/FlossWare/.github/tree/main/docs/development)** | Getting started, contributing, coding standards |

---

## Getting Started

**New to FlossWare?**

1. Read this README
2. Explore **[ARCHITECTURE.md](https://github.com/FlossWare/.github/blob/main/ARCHITECTURE.md)**
3. Clone **[consensus-ai](https://github.com/FlossWare/consensus-ai)**
4. Run the example workflows
5. Explore the remaining repositories

```bash
# Clone the orchestration framework
git clone https://github.com/FlossWare/consensus-ai.git

# Or explore the architecture documentation
git clone https://github.com/FlossWare/.github.git
```

See **[Getting Started](https://github.com/FlossWare/.github/blob/main/docs/development/getting_started.md)** for the full setup guide.

---

## Contributing

FlossWare is open to contributions from developers, AI practitioners, and infrastructure engineers. See **[Contributing](https://github.com/FlossWare/.github/blob/main/docs/development/contributing.md)** for guidelines.

---

## Core Principle

**The models will change. The orchestration endures.**

---

*Built with multi-AI consensus. Reviewed by adversarial panels. Evolved by genetic algorithms.*
