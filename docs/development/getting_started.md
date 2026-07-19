# Getting Started

This guide walks you through setting up your development environment and making your first interactions with the distributed LLM orchestration framework.

---

## Prerequisites

- **Node.js 18+** for running workflows
- **Python 3.10+** for scrapers, GA optimizers, and analysis tools
- **Git** for source control
- **SSH access** to at least one fleet node (for deployment testing)
- **curl** or similar HTTP client for API interaction

---

## Quick Start

Clone the repository and explore the project structure:

```bash
git clone https://github.com/FlossWare/consensus-ai.git
cd consensus-ai
npm install
```

Read `docs/architecture/ARCHITECTURE.md` for an overview of how the controller, fleet workers, pipelines, and datastores fit together.

---

## Making Your First Consensus Request

The simplest way to interact with the system is a direct multi-model query via the orchestrator API:

```bash
curl -X POST http://aio-01:5000/consensus \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Explain the CAP theorem in two sentences.",
    "models": ["opus", "sonnet", "haiku"],
    "strategy": "majority_vote"
  }'
```

Response:

```json
{
  "consensus": "The CAP theorem states that a distributed system can provide at most two of three guarantees: consistency, availability, and partition tolerance. In practice, since network partitions are unavoidable, systems must choose between consistency and availability during a partition.",
  "agreement_score": 0.92,
  "model_responses": [
    {"model": "opus", "response": "..."},
    {"model": "sonnet", "response": "..."},
    {"model": "haiku", "response": "..."}
  ],
  "strategy": "majority_vote"
}
```

The `agreement_score` indicates how closely the models agreed. Scores above 0.85 generally indicate strong consensus.

---

## Running a Workflow

Workflows are multi-phase, multi-model task pipelines. Run one from the command line:

```bash
node workflows/deep-research.mjs --topic "circuit breaker patterns in distributed systems"
```

Workflows follow a standard structure:

```javascript
export const meta = {
  name: 'my-workflow',
  description: 'One-line description of what this workflow does',
  phases: [
    { title: 'Research', detail: 'Gather information from multiple models' },
    { title: 'Synthesize', detail: 'Combine findings into a coherent result' }
  ]
};

export default async function({ phase, parallel, agent, log }) {
  phase('Research');
  const findings = await parallel([
    agent('Research aspect A'),
    agent('Research aspect B'),
    agent('Research aspect C')
  ]);

  phase('Synthesize');
  const result = await agent('Synthesize these findings into a report');

  return { findings, result };
}
```

All workflow executions are automatically tracked in the `workflow.*` schema for later analysis.

---

## Adding a New Scraper

Scrapers feed documents into the ingestion pipeline. To add one:

1. Create a new Python file in `scrapers/`:

```python
from base_scraper import BaseScraper

class MyDocsScraper(BaseScraper):
    """Scraper for MyDocs documentation site."""

    def __init__(self):
        super().__init__(
            name="mydocs",
            base_url="https://docs.example.com",
            output_dir="/home/claude/workers/scrapers/mydocs"
        )

    def get_urls(self):
        """Return list of URLs to scrape."""
        return [
            "https://docs.example.com/guide",
            "https://docs.example.com/api-reference",
        ]

    def parse(self, url, html):
        """Extract content from a page."""
        # Return structured document
        return {
            "title": self.extract_title(html),
            "content": self.extract_body(html),
            "url": url
        }

if __name__ == "__main__":
    MyDocsScraper().run()
```

2. Deploy to a worker and launch it (see the Deployment Guide).
3. The ingestion pipeline picks up new documents automatically.

---

## Exploring the Knowledge Base

Query the search API to find documents in the knowledge base:

```bash
curl "http://aio-01:5000/search?q=circuit+breaker+pattern&limit=10"
```

For vector similarity search:

```bash
curl -X POST http://aio-01:5000/search/similar \
  -H "Content-Type: application/json" \
  -d '{"text": "how to implement retry logic with exponential backoff", "limit": 5}'
```

---

## Understanding Model Routing

The system uses Thompson Sampling to route tasks to models. Each model maintains a beta distribution (alpha, beta) representing its observed success rate.

```bash
# View current strategy performance
curl http://aio-01:5000/api/strategies
```

This returns per-model statistics:

| Field        | Meaning                                    |
|--------------|--------------------------------------------|
| `strategy`   | Model name                                 |
| `avg_reward`  | Historical average quality score           |
| `successes`  | Number of successful executions            |
| `failures`   | Number of failed executions                |

Models with higher `avg_reward` receive more traffic, but the Thompson Sampling algorithm maintains exploration -- occasionally routing tasks to less-proven models to gather data.

---

## Development Environment Setup

### Required Dependencies

```bash
# JavaScript dependencies
npm install

# Python dependencies
pip3 install sentence-transformers psycopg2-binary requests
```

### Environment Variables

No environment variables are required for basic development. The shared adapters (`postgres-adapter.js`, `workflow-storage-adapter.js`) handle connection configuration internally.

### Running Tests

```bash
npm test
```

---

## Key API Endpoints

These are the endpoints you will use most often during development:

| Endpoint                     | Method | Purpose                          |
|------------------------------|--------|----------------------------------|
| `/health`                    | GET    | System health check              |
| `/consensus`                 | POST   | Multi-model consensus query      |
| `/fleet/status`              | GET    | Fleet worker status              |
| `/queue/status`              | GET    | Queue depths and counts          |
| `/search`                    | GET    | Full-text knowledge search       |
| `/search/similar`            | POST   | Vector similarity search         |
| `/api/strategies`            | GET    | Model routing performance        |
| `/api/executions`            | GET    | Recent execution logs            |
| `/providers/health`          | GET    | API provider health status       |

---

## Next Steps

- Read the [Architecture Guide](../architecture/ARCHITECTURE.md) for the full system design.
- Read the [Contributing Guide](contributing.md) before submitting changes.
- Read the [Coding Standards](coding_standards.md) for code conventions.
- Explore existing workflows in `workflows/` for patterns to follow.
