# FlossWare

**Multi-model AI engineering. 200+ LLMs orchestrated to build, review, and evolve software.**

FlossWare coordinates 200+ language models across 8+ providers through consensus, adversarial verification, and evolutionary optimization. Models write code, independent panels review it with zero overlap, and genetic algorithms evolve better configurations over time.

---

## How It Works

- **Multi-AI code review** — 4-phase pipeline (Review, Meta-Review, Fix, Verify) where review and meta-review panels share zero models, preventing self-confirmation bias
- **Knowledge-grounded development** — 116+ scrapers feed a pipeline (ingest, chunk, embed, graph) producing 381K+ documents, 3.9M+ chunks, and 487K+ embeddings for RAG
- **Evolutionary optimization** — 7 GA optimizers evolve model routing, team composition, RAG parameters, and prompt templates using real execution history
- **Distributed fleet** — 9 nodes (1 controller + 8 workers) across x86_64, ARM64, and ARMv7, all models via free-tier APIs

```
┌─────────────────────────────┐
│     Clients / Agents        │
└──────────────┬──────────────┘
               │ REST API
┌──────────────▼──────────────┐
│   Orchestrator (Flask API)  │
│  Routing · Consensus · GA   │
└──────┬──────────┬───────────┘
       │ SSH      │ API
┌──────▼──────┐ ┌─▼───────────┐
│ Worker Fleet│ │ 200+ LLMs   │
│ (8 nodes)   │ │ (8+ provs)  │
└──────┬──────┘ └─────────────┘
       │
┌──────▼──────────────────────┐
│   Knowledge Platform        │
│ PostgreSQL · Redis · OrientDB│
│ Vectors · Queues · Graph    │
└─────────────────────────────┘
```

---

## Repositories

FlossWare has 40 repositories across four categories.

### AI Engineering Infrastructure

| Repository | Description |
|------------|-------------|
| [consensus-ai](https://github.com/FlossWare/consensus-ai) | Multi-AI orchestration library — 5 consensus strategies |
| [knowledge-ai](https://github.com/FlossWare/knowledge-ai) | Universal knowledge ingestion from any documentation format |
| [skills-ai](https://github.com/FlossWare/skills-ai) | Executable workflows for the AI ecosystem |
| [vectordb-ai](https://github.com/FlossWare/vectordb-ai) | Universal vector database adapter — 9 backends, no vendor lock-in |
| [semantic-search-ai](https://github.com/FlossWare/semantic-search-ai) | Hybrid search, reranking, filtering for AI applications |
| [netbeans-plugins](https://github.com/FlossWare/netbeans-plugins) | NetBeans IDE plugins for Claude, Gemini, and ChatGPT |

### Java Infrastructure

| Repository | Description |
|------------|-------------|
| [commons-java](https://github.com/FlossWare/commons-java) | Shared utilities — SOAP clients, string operations, file handling |
| [platform-java](https://github.com/FlossWare/platform-java) | Multi-application isolation — classloaders, thread pools, security |
| [curses-java](https://github.com/FlossWare/curses-java) | Terminal UI library — 29 AWT-like widgets, ncurses backend |
| [collections-java](https://github.com/FlossWare/collections-java) | Collections backed by files, networking, and other storage |
| [classloader-java](https://github.com/FlossWare/classloader-java) | Universal ClassLoader supporting 30+ protocols |
| [remote-java](https://github.com/FlossWare/remote-java) | RPC framework — multi-format serialization (JSON/XML/YAML/MessagePack) |
| [container-java](https://github.com/FlossWare/container-java) | Container/orchestration abstraction (Kubernetes, Docker, Hazelcast) |
| [cloudstorage-java](https://github.com/FlossWare/cloudstorage-java) | Cloud storage abstraction (S3, Azure Blob, GCS, Drive, Dropbox) |
| [filetransfer-java](https://github.com/FlossWare/filetransfer-java) | File transfer abstraction (SFTP, WebDAV, SMB/CIFS, FTP) |
| [messaging-java](https://github.com/FlossWare/messaging-java) | Messaging/cache abstraction (Kafka, RabbitMQ, Redis) |
| [vcs-java](https://github.com/FlossWare/vcs-java) | Version control abstraction (Git) |
| [encrypt-java](https://github.com/FlossWare/encrypt-java) | AES-256-GCM encryption library |
| [eventbus-java](https://github.com/FlossWare/eventbus-java) | Event bus and service registry |
| [threadpool-java](https://github.com/FlossWare/threadpool-java) | Managed thread pools with monitoring and graceful shutdown |
| [resource-monitor-java](https://github.com/FlossWare/resource-monitor-java) | Resource usage tracking and quota enforcement |
| [fs-watcher-java](https://github.com/FlossWare/fs-watcher-java) | Filesystem watcher with debouncing |
| [nexus-java](https://github.com/FlossWare/nexus-java) | Nexus Repository Manager CLI/GUI — search, filtering, analytics |
| [diskwipe-java](https://github.com/FlossWare/diskwipe-java) | Secure disk space wiping |
| [build-tools](https://github.com/FlossWare/build-tools) | Automated code quality and refactoring tools |

### Systems Software

| Repository | Description |
|------------|-------------|
| [pxe-os](https://github.com/FlossWare/pxe-os) | Cross-OS PXE boot provisioning (Linux, BSD, Windows) |
| [virt-os](https://github.com/FlossWare/virt-os) | Minimal virtualization OS based on Tiny Core Linux |
| [virt-os-examples](https://github.com/FlossWare/virt-os-examples) | Ready-to-deploy templates for VirtOS microservices |
| [tftp-os](https://github.com/FlossWare/tftp-os) | TFTP-based firmware provisioning — standalone base for PxeOS |
| [cobbler](https://github.com/FlossWare/cobbler) | Cobbler templates for RHEL/Fedora, Debian/Ubuntu, FreeBSD |
| [notion2config](https://github.com/FlossWare/notion2config) | Generate system configs from Notion databases |
| [de-converter](https://github.com/FlossWare/de-converter) | Convert desktop environment configs to lightweight window managers |

### Applications

| Repository | Description |
|------------|-------------|
| [hotspot-android](https://github.com/FlossWare/hotspot-android) | Free Android hotspot app for internet outages |
| [samsung-galaxy-j7](https://github.com/FlossWare/samsung-galaxy-j7) | Transform Galaxy J7 into a mini Linux computer |
| [civilization-simulator-java](https://github.com/FlossWare/civilization-simulator-java) | Alternate history civilization simulator |
| [curses-themes](https://github.com/FlossWare/curses-themes) | Lightweight theme support for Python curses |
| [flossware-nexus](https://github.com/FlossWare/flossware-nexus) | Desktop, Android, iOS clients for Nexus Repository Manager |
| [flossware-tftpos](https://github.com/FlossWare/flossware-tftpos) | Desktop, mobile, web clients for TFTP-based firmware provisioning |

### Documentation

| Repository | Description |
|------------|-------------|
| [.github](https://github.com/FlossWare/.github) | Organization profile and architecture documentation |
| [FlossWare](https://github.com/FlossWare/FlossWare) | Generated documentation (Javadocs) |

---

## Documentation

All documentation lives in [FlossWare/.github](https://github.com/FlossWare/.github). Direct links to each guide:

### Architecture

| Guide | Description |
|-------|-------------|
| [Architecture Guide](https://github.com/FlossWare/.github/blob/main/ARCHITECTURE.md) | Complete system architecture — all subsystems in one reference |
| [Orchestration](https://github.com/FlossWare/.github/blob/main/docs/architecture/orchestration.md) | How the orchestrator routes tasks, manages workers, and coordinates models |
| [Consensus](https://github.com/FlossWare/.github/blob/main/docs/architecture/consensus.md) | Multi-model consensus strategies and voting mechanisms |
| [Fleet](https://github.com/FlossWare/.github/blob/main/docs/architecture/fleet.md) | Distributed fleet topology, worker tiers, and SSH execution |
| [Routing](https://github.com/FlossWare/.github/blob/main/docs/architecture/routing.md) | Model routing policies and provider fallback chains |
| [Design Philosophy](https://github.com/FlossWare/.github/blob/main/docs/philosophy.md) | Engineering principles and design reasoning |

### Knowledge Pipeline

| Guide | Description |
|-------|-------------|
| [Scraping](https://github.com/FlossWare/.github/blob/main/docs/knowledge/scraping.md) | 116+ scrapers, BaseScraper pattern, rate limiting |
| [Chunking](https://github.com/FlossWare/.github/blob/main/docs/knowledge/chunking.md) | Text chunking strategies (512 tokens, 37% overlap) |
| [Embeddings](https://github.com/FlossWare/.github/blob/main/docs/knowledge/embeddings.md) | Vector embeddings with all-mpnet-base-v2 (768-dim) |
| [Graph](https://github.com/FlossWare/.github/blob/main/docs/knowledge/graph.md) | OrientDB knowledge graph for entity relationships |

### Databases

| Guide | Description |
|-------|-------------|
| [PostgreSQL](https://github.com/FlossWare/.github/blob/main/docs/databases/postgres.md) | Primary datastore with pgvector for similarity search |
| [Redis](https://github.com/FlossWare/.github/blob/main/docs/databases/redis.md) | Queue-based pipeline and caching layer |
| [OrientDB](https://github.com/FlossWare/.github/blob/main/docs/databases/orientdb.md) | Graph database for infrastructure and knowledge relationships |

### Learning & Optimization

| Guide | Description |
|-------|-------------|
| [Genetic Algorithms](https://github.com/FlossWare/.github/blob/main/docs/learning/genetic_algorithms.md) | 7 GA optimizers for routing, teams, RAG, prompts |
| [Thompson Sampling](https://github.com/FlossWare/.github/blob/main/docs/learning/thompson_sampling.md) | Bayesian bandit for model selection |

### Operations

| Guide | Description |
|-------|-------------|
| [Deployment](https://github.com/FlossWare/.github/blob/main/docs/operations/deployment.md) | Ansible-based fleet deployment |
| [Monitoring](https://github.com/FlossWare/.github/blob/main/docs/operations/monitoring.md) | Prometheus, Grafana, and feedback loop monitoring |
| [Scaling](https://github.com/FlossWare/.github/blob/main/docs/operations/scaling.md) | Horizontal and vertical scaling guides |

### Development

| Guide | Description |
|-------|-------------|
| [Getting Started](https://github.com/FlossWare/.github/blob/main/docs/development/getting_started.md) | Developer setup and first workflow |
| [Contributing](https://github.com/FlossWare/.github/blob/main/docs/development/contributing.md) | Contribution guidelines and code review process |
| [Coding Standards](https://github.com/FlossWare/.github/blob/main/docs/development/coding_standards.md) | Style guide and conventions |

---

## Getting Started

```bash
git clone https://github.com/FlossWare/consensus-ai.git
```

See the [Getting Started](https://github.com/FlossWare/.github/blob/main/docs/development/getting_started.md) guide for full setup, or explore the [Architecture Guide](https://github.com/FlossWare/.github/blob/main/ARCHITECTURE.md) for the complete system reference.

---

## Contributing

See [Contributing](https://github.com/FlossWare/.github/blob/main/docs/development/contributing.md) for guidelines. All code goes through multi-AI review with zero-overlap panels.

---

*Built with multi-model engineering. Reviewed by adversarial panels. Evolved by genetic algorithms.*
