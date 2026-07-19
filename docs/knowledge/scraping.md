# Web Scraping

This document describes the web scraping subsystem: how documents enter the knowledge pipeline, the scraper architecture, fleet distribution, and integration with the downstream processing stages.

## Why We Scrape

Large language models are frozen at their training cutoff. They cannot know about a CVE
published last week, a Kubernetes flag deprecated in the latest release, or a PostgreSQL
optimization technique documented in a blog post six months after training ended. When a
model encounters a question outside its training data, it does one of two things: refuses
to answer, or confabulates -- producing a fluent, confident, and wrong response.

Retrieval-Augmented Generation (RAG) solves this by injecting verified, up-to-date source
material into the model's context window at query time. Instead of relying on parametric
memory (what the model "learned" during training), the system retrieves relevant document
chunks from a vector store and includes them as grounding context. The model generates its
answer *from* the retrieved sources rather than *from* its training weights.

This only works if the vector store contains high-quality, current, domain-specific
content. That is what the scraping layer provides.

**What scraping enables:**

- **Factual grounding.** Every answer can cite specific documentation rather than relying
  on parametric recall. A question about Terraform provider configuration returns the
  actual Terraform docs, not a training-era approximation.
- **Currency.** The corpus is continuously refreshable. When OpenWrt releases a new
  version, re-running its scraper brings the knowledge base up to date without retraining
  any model.
- **Domain depth.** The 200+ models in the fleet were trained as generalists. Scraping
  315,000+ documents across 12 specialized domains (chip design, embedded systems,
  culinary science, etc.) gives them domain expertise they would not otherwise have.
- **Hallucination reduction.** When the model's context window contains the actual
  documentation, it has less reason to fabricate. Empirically, RAG-grounded responses
  produce fewer factual errors than ungrounded ones, particularly on niche topics where
  the model's training data is sparse.
- **Auditability.** Every chunk in the vector store traces back to a source URL. When a
  user questions an answer, the system can show exactly which documents informed it.

Without the scraping layer, the system would be limited to whatever the models memorized
during training -- static, aging, and unverifiable. With it, the system has a continuously
growing, independently verifiable knowledge base that any model in the fleet can draw from.

## Overview

The scraping layer is responsible for ingesting structured and semi-structured web content into the knowledge pipeline. It currently comprises 88+ scrapers covering 12 knowledge domains, producing a corpus of 315,000+ documents. Scrapers run as single-pass Python processes on fleet workers, enqueuing results into a Redis-backed pipeline via the controller's REST API.

## Knowledge Domains

| Domain | Example Scrapers | Example Sources |
|--------|-----------------|-----------------|
| Programming | Python, Rust, Go, TypeScript, C++, Ruby, Haskell, Swift, PHP | docs.python.org, doc.rust-lang.org, cppreference.com |
| Frameworks | React, Next.js, Django, FastAPI, Vue.js | react.dev, nextjs.org, djangoproject.com |
| Cloud / DevOps | Kubernetes, Docker, Terraform, Ansible, Helm, ArgoCD, Istio | kubernetes.io, docs.docker.com, terraform.io |
| Systems | Arch Wiki, Gentoo, NixOS, FreeBSD, Linux Kernel, systemd | wiki.archlinux.org, wiki.gentoo.org, kernel.org |
| Data / ML | PyTorch, scikit-learn, pandas, HuggingFace | pytorch.org, scikit-learn.org, huggingface.co |
| Databases | PostgreSQL, MongoDB, MySQL, Redis, Elasticsearch | postgresql.org, docs.mongodb.com, redis.io |
| Security | MITRE ATT&CK, OWASP | attack.mitre.org, owasp.org |
| Web Standards | MDN Web Docs | developer.mozilla.org |
| Science | NASA, HyperPhysics, LibreTexts (physics, math, bio), MedlinePlus | nasa.gov, hyperphysics.phy-astr.gsu.edu |
| Chip Design | ChipVerify, ASIC World, Nandland, FPGA4Fun, ZipCPU, WikiChip | chipverify.com, asic-world.com, nandland.com |
| Culinary | Serious Eats, Epicurious, BBC Good Food, Cooking for Engineers, Wikibooks Cookbook | seriouseats.com, epicurious.com |
| Embedded | Arduino, Raspberry Pi, ESP32, OpenWrt, FreshTomato | arduino.cc, raspberrypi.com, openwrt.org |

## BaseScraper Pattern

All scrapers inherit from `BaseScraper` (defined in `scrapers/scraper_base.py`). The contract is minimal: subclass, define your sources, implement `scrape()`, and call `self.save_item()` for each document. The base class handles disk persistence, batch enqueueing to the Redis pipeline, statistics tracking, graceful shutdown via signal handlers, and structured logging.

```
                                +--------------+
                                | BaseScraper  |
                                |--------------|
                                | save_item()  |
                                | fetch_url()  |
                                | fetch_json() |
                                | make_id()    |
                                | run()        |
                                +------+-------+
                                       |
              +------------------------+------------------------+
              |                        |                        |
     +--------+--------+    +---------+--------+    +----------+--------+
     | PythonScraper    |    | ArchWikiScraper  |    | KubernetesScraper |
     | SOURCES = {...}  |    | _enumerate_pages |    | SOURCES = {...}   |
     | scrape()         |    | scrape()         |    | scrape()          |
     +-----------------+    +------------------+    +-------------------+
```

### Implementing a Scraper

A typical scraper defines a `SOURCES` dictionary mapping URLs to titles, then iterates through them in `scrape()`:

```python
from scraper_base import BaseScraper

class ExampleScraper(BaseScraper):
    SOURCES = {
        "reference": {
            "pages": {
                "https://example.com/docs/intro": "Introduction",
                "https://example.com/docs/api":   "API Reference",
                "https://example.com/docs/faq":   "FAQ",
            },
        },
    }

    def __init__(self, base_dir, source_key=None):
        super().__init__("example", base_dir, interval_seconds=3600)
        self.source_key = source_key

    def scrape(self):
        count = 0
        for key, config in self.SOURCES.items():
            for url, title in config["pages"].items():
                content = self.fetch_url(url)
                if content and len(content) > 500:
                    text = self._strip_html(content)
                    item_id = self.make_id(url)
                    if self.save_item(item_id, {
                        "title": title,
                        "content": text[:50000],
                        "url": url,
                        "category": "example",
                        "type": "documentation",
                    }):
                        count += 1
                time.sleep(1.5)  # rate limiting
        return count
```

When `save_item()` is called, two things happen:

1. The document is written to disk as JSON (deduplicated by content hash).
2. The document is batched (default batch size: 20) and enqueued to the Redis `store` queue via `POST /queue/enqueue` on the controller's REST API.

### What save_item Triggers

```
save_item(id, data)
    |
    +---> write JSON to disk (scraped-data/<source>/<id>.json)
    |
    +---> _enqueue_to_pipeline(data)
              |
              +---> append to internal batch buffer
              |
              +---> if batch >= 20: _flush_enqueue()
                        |
                        +---> POST /queue/enqueue  { items: [...] }
                                   |
                                   +---> Redis LPUSH store:<priority>
```

## MediaWiki API Enumeration

For wiki-based sources (Arch Wiki, Gentoo Wiki, OpenWrt), static URL lists are impractical -- these sites have thousands of pages and the catalog changes over time. Instead, these scrapers use the MediaWiki `allpages` API to dynamically enumerate content:

```python
def _enumerate_pages(self):
    pages = []
    apcontinue = None

    while len(pages) < self.MAX_PAGES and self.running:
        params = {
            "action": "query",
            "list": "allpages",
            "aplimit": "50",
            "apnamespace": "0",  # main namespace only
            "format": "json",
        }
        if apcontinue:
            params["apcontinue"] = apcontinue

        data = self.fetch_json(f"{self.API_URL}?{urlencode(params)}")
        # ... collect page titles, advance cursor ...

        apcontinue = data.get("continue", {}).get("apcontinue")
        if not apcontinue:
            break
        time.sleep(0.5)

    return pages
```

The `apcontinue` cursor provides stable pagination through the entire page catalog. Namespace filters (`apnamespace=0`) exclude talk pages, user pages, templates, and other non-content namespaces. The Arch Wiki scraper further filters by skip patterns (regex matching `Talk:`, `User:`, `Template:`, etc.) and detects redirect/disambiguation pages to avoid ingesting stubs.

## Rate Limiting

Scrapers enforce per-domain politeness:

- **User-Agent:** All requests use `ResearchScraper/1.0 (academic research)` to identify the bot.
- **Per-request delays:** 1.0-1.5 seconds between fetches (configurable per scraper). Wiki API enumeration calls use a lighter 0.5-second delay since they transfer less data.
- **Timeout:** 30-second default per HTTP request.
- **Max content length:** Documents are truncated to 50,000 characters at save time, preventing runaway memory usage on very large pages.

## Fleet Distribution

Scrapers run on fleet workers (server-01, server-02, server-03, laptop-01, etc.), never on the controller (aio-01). The controller orchestrates execution via SSH:

```
Controller (aio-01)                    Workers
+-----------------+          +------------------------+
| Dispatch scraper|  --SSH-->| server-01: python.py   |
| via SSH to idle |  --SSH-->| server-02: archwiki.py  |
| workers         |  --SSH-->| server-03: kubernetes.py|
+-----------------+          +------------------------+
        |                              |
        |                    POST /queue/enqueue
        |                              |
        v                              v
+-------------------------------------------+
|          Redis (aio-01:6379)              |
|  store:5  [ doc, doc, doc, ... ]          |
+-------------------------------------------+
```

Each worker independently runs its assigned scraper, which calls `save_item()` to enqueue documents back to the central Redis pipeline on the controller. Workers operate independently and do not coordinate with each other -- deduplication happens at the storage layer via content hashing.

## Performance: Decoupled Pipeline

The original monolithic approach ran scraping, storage, chunking, and embedding in a single process. This produced approximately 599 documents per hour because embedding (the compute-bound stage) throttled the entire chain.

The decoupled architecture separates concerns:

```
Scraping (network-bound)       Processing (compute-bound)
+-------------------+         +--------------------+
| Fetch HTML pages  |         | Store documents    |
| ~50ms per page    | ------> | Chunk text         |
| + rate limit      | Redis   | Generate embeddings|
| = ~1.5s per doc   | Queue   | = ~2-5s per doc    |
+-------------------+         +--------------------+
```

**Why this matters:** Scraping is network-bound -- the bottleneck is the rate-limit delay between requests, not CPU. Embedding is compute-bound -- each document requires a forward pass through a transformer model. When these run in the same process, the embedding stage (2-5 seconds per document) blocks the scraping stage (1.5 seconds per document including rate limiting). Decoupling them via Redis allows scraping to run at full network speed while a separate pool of embed workers processes the backlog asynchronously.

**Result:** 7.8x throughput improvement, from 599 to 4,700 documents per hour.

## Store Workers

Store workers are lightweight processes that drain the Redis `store` queue and persist documents to PostgreSQL. They follow a claim-process-complete protocol:

1. `POST /queue/fetch/store` -- claim a batch of items from the queue (default batch size: 5).
2. `POST /store` -- write each document to the `knowledge.documents` table.
3. `POST /queue/complete` -- mark the item as processed, removing it from the queue.

If storage fails, `POST /queue/fail` returns the item to the queue with an incremented retry counter (max 3 retries). Store workers can run on any node since they interact entirely through the REST API.

Completing the store stage for a document auto-enqueues it to the `chunk` stage, which in turn auto-enqueues to the `embed` stage upon completion. This forms a linear pipeline:

```
scrape --> [Redis: store queue] --> store --> [Redis: chunk queue] --> chunk --> [embed]
```

## Error Handling

- **Non-fatal pipeline failures:** If the REST API is unreachable during enqueue, `_flush_enqueue()` logs a warning and continues scraping. Documents are still persisted to disk and can be re-ingested later.
- **Graceful shutdown:** `SIGTERM` and `SIGINT` trigger `_shutdown()`, which flushes any pending enqueue batch before exiting. This prevents data loss during controlled restarts.
- **Deduplication:** `save_item()` skips documents whose content-hash file already exists on disk. At the database level, the `knowledge.documents` table enforces uniqueness on `(url, source)`.
