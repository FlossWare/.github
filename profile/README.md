# FlossWare

**AI-powered engineering. Multi-model collaboration. Software evolution.**

FlossWare builds and evolves complex software using coordinated AI engineering systems. Rather than prompting a single model and shipping the result, FlossWare treats 200+ language models as engineering tools -- orchestrated through consensus, adversarial verification, and evolutionary optimization -- to produce software that no single model could build or review alone.

The AI is the engineering method. The software is the product.

---

## The Problem

Software built by one AI model inherits that model's blind spots. Code reviewed by the same model that wrote it confirms its own assumptions. Parameters tuned by hand reflect the operator's guesses, not measured outcomes.

## How FlossWare Builds Software

Instead of trusting one model, FlossWare coordinates 200+ models from 8+ providers (Anthropic, OpenAI, Google, Groq, Cerebras, DeepSeek, and others) across a distributed fleet. Models write code, then independent panels review it with zero overlap. Separate panels adversarially challenge those reviews. Genetic algorithms evolve better configurations over time. The result is software built through independent verification at every stage.

---

## What Gets Built

The AI engineering systems described here produce real software. These are not demos or chatbot wrappers -- they are standalone projects that solve specific problems:

| Project | What It Does |
|---------|-------------|
| **[PxeOS](https://github.com/FlossWare/PxeOS)** | Cross-OS PXE boot provisioning (Linux, BSD, Windows) |
| **[VirtOS](https://github.com/FlossWare/VirtOS)** | Minimal virtualization OS based on Tiny Core Linux |
| **[consensus-ai](https://github.com/FlossWare/consensus-ai)** | Multi-AI orchestration library with 5 consensus strategies |
| **[knowledge-ai](https://github.com/FlossWare/knowledge-ai)** | Universal knowledge ingestion from any documentation format |
| **[skills-ai](https://github.com/FlossWare/skills-ai)** | Executable workflows for the AI ecosystem |
| **[commons-java](https://github.com/FlossWare/commons-java)** | Shared utilities for SOAP clients, string operations, file handling |
| **[platform-java](https://github.com/FlossWare/platform-java)** | Multi-application isolation -- classloaders, thread pools, security |
| **[curses-java](https://github.com/FlossWare/curses-java)** | Terminal UI library with 29 AWT-like widgets and ncurses backend |

These projects are written, reviewed, and evolved using the multi-model engineering pipeline. The AI doesn't just assist -- it writes the code, reviews its own work through adversarial panels, and evolves its approach based on execution history.

---

## Engineering Philosophy

**AI models are components, not the product.** The 200+ models are interchangeable tools in an engineering pipeline. When a model improves or a provider changes pricing, the system routes around it. No single model is essential.

**Consensus and orchestration are engineering tools.** Multi-model consensus exists to catch bugs that single-model review misses. Adversarial verification exists to prevent self-confirmation. These are quality engineering practices implemented through AI, not AI features implemented for their own sake.

**The goal is better software, not better chatbots.** Every system described here -- scraping, embedding, routing, evolution -- serves one purpose: produce software that works correctly and survives independent scrutiny.

**Evolution over configuration.** Don't hand-tune parameters. Let genetic algorithms search the configuration space using real execution history as fitness data.

**Interfaces over implementations.** Every component communicates through REST APIs. The orchestrator doesn't know if a worker is a server or a Raspberry Pi. Any component can be replaced without touching the others.

For the full design philosophy, see **[Design Philosophy](https://github.com/FlossWare/.github/blob/main/docs/philosophy.md)**.

---

## How the Engineering Pipeline Works

### Multi-AI Code Review

The signature engineering practice. Code changes pass through four phases -- **Review**, **Meta-Review**, **Fix**, **Verify** -- where review and meta-review panels share **zero models** to prevent self-confirmation bias. Each phase uses a different arbiter. External reviewers (Grok, ChatGPT, Notebook LLM) provide independent adversarial critique outside the orchestrator entirely.

### Knowledge-Grounded Development

88+ scrapers collect documents across 12+ domains. A 4-stage async pipeline (store, chunk, embed, graph) converts raw content into searchable vector embeddings and graph-linked knowledge. This grounds AI-generated code in real documentation -- 315,000+ documents, 705,000+ embeddings, 0.4ms vector search -- reducing hallucination and ensuring generated code follows actual API patterns.

### Evolutionary Optimization

Seven GA optimizers evolve model routing, team composition, RAG parameters, prompt templates, and workflow configurations using real execution history as fitness data. The system doesn't stay static -- it measures what works and evolves toward better outcomes.

### Distributed Execution

9 nodes (1 controller + 8 workers) spanning three CPU architectures (x86_64, ARM64, ARMv7) connected via SSH. All 200+ models accessed via free-tier APIs. Circuit breakers, provider fallback, and rate limiting ensure the engineering pipeline stays operational.

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

For the complete architecture, see the **[Architecture Guide](https://github.com/FlossWare/.github/blob/main/ARCHITECTURE.md)**.

---

## Repository Map

FlossWare consists of 37 repositories. The AI engineering pipeline builds and maintains software across all of them.

### AI Engineering Infrastructure

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

### Systems Software

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
| **[Architecture Guide](https://github.com/FlossWare/.github/blob/main/ARCHITECTURE.md)** | Complete system architecture (20+ pages) |
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
2. Explore the **[Architecture Guide](https://github.com/FlossWare/.github/blob/main/ARCHITECTURE.md)**
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

**The models will change. The software endures.**

---

*Built with multi-model engineering. Reviewed by adversarial panels. Evolved by genetic algorithms.*
