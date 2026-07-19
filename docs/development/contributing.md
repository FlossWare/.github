# Contributing Guide

Thank you for your interest in contributing to the FlossWare distributed LLM orchestration framework. This guide explains how to contribute effectively.

---

## How to Contribute

1. **Find an issue.** Browse open issues on GitHub. Issues labeled `good-first-issue` are suitable for new contributors.
2. **Discuss your approach.** Comment on the issue with your proposed approach before starting work. This avoids duplicate effort and ensures alignment with project direction.
3. **Implement.** Follow the coding standards and patterns described below.
4. **Submit a pull request.** Include a clear description, test coverage, and documentation updates as needed.

---

## Code Review Process

All changes go through the multi-AI review pipeline. This is not a manual gate -- it is an automated system that runs during PR evaluation.

The review pipeline has four phases:

1. **Review** -- A panel of models analyzes the change for correctness, security, and style issues.
2. **Meta-review** -- A separate panel (with zero model overlap from the review panel) adversarially validates the review findings.
3. **Fix** -- Identified issues are addressed.
4. **Verify** -- A final verification pass confirms the fixes are correct.

Each phase uses a different arbiter model to prevent self-confirmation bias. The review and meta-review panels must have zero model overlap.

Maintainers review the automated findings alongside the code diff. Human judgment remains the final arbiter.

---

## Branch Workflow

- **Main branch:** `main` is the integration branch. It should always be in a working state.
- **Feature branches:** Create a branch per feature or fix. Name it descriptively: `add-openwrt-scraper`, `fix-circuit-breaker-timeout`, `improve-embedding-throughput`.
- **Git worktrees:** For parallel work on multiple features, use git worktrees rather than switching branches:

```bash
git worktree add ../my-feature-worktree -b my-feature
cd ../my-feature-worktree
# Work in isolation, then clean up
git worktree remove ../my-feature-worktree
```

---

## Pull Request Guidelines

A good PR includes:

- **Clear title** (under 70 characters) summarizing the change.
- **Description** explaining what changed and why.
- **Test coverage** for new functionality.
- **Documentation updates** if the change affects user-facing behavior, API endpoints, or configuration.
- **Small scope** -- one logical change per PR. Large PRs are harder to review and more likely to introduce regressions.

PR template:

```markdown
## Summary

- Added BaseScraper subclass for OpenWrt documentation
- Configured pipeline to handle OpenWrt's nested page structure

## Test Plan

- [ ] Scraper produces valid documents for 5 sample URLs
- [ ] Documents pass through chunking and embedding pipeline
- [ ] Search returns relevant results from scraped content
```

---

## Areas Where Contributions Are Welcome

### New Scrapers

The knowledge base grows through scrapers. Follow the `BaseScraper` pattern (see the Getting Started guide). Particularly useful:

- Documentation sites for open-source projects
- Technical reference materials
- Standards and specification documents

### New GA Optimizers

The genetic algorithm framework optimizes system parameters. To add a new optimizer:

1. Define a chromosome structure representing the parameters to optimize.
2. Implement a fitness function that evaluates chromosome quality.
3. Connect to the REST API for persistence.
4. Use the core GA engine (`learning/ga_engine.py`) for crossover, mutation, and selection.

See existing optimizers in `tools/` for reference implementations.

### New Consensus Strategies

The routing layer supports pluggable consensus strategies beyond majority vote. Ideas:

- Weighted voting based on model specialization
- Hierarchical consensus (fast models filter, strong models decide)
- Domain-specific routing (code tasks to code-specialized models)

### Documentation Improvements

Clear documentation helps everyone. Contributions that improve clarity, fix errors, or add missing information are always welcome.

### Performance Optimizations

The system processes hundreds of thousands of documents. Optimizations in the pipeline stages (chunking, embedding, search) have measurable impact. Profile before optimizing -- identify the actual bottleneck using queue metrics.

### New Integrations

The system communicates via REST API, making it straightforward to integrate with external tools:

- CI/CD pipelines
- Chat platforms
- Monitoring systems
- Other knowledge sources

---

## Development Setup

```bash
git clone https://github.com/FlossWare/consensus-ai.git
cd consensus-ai
npm install
pip3 install -r requirements.txt
```

Run tests before submitting:

```bash
npm test
```

---

## Communication

- **GitHub Issues:** Bug reports, feature requests, and design discussions.
- **Pull Requests:** Code review and implementation discussion.
- **Issue Comments:** Propose your approach before starting work on non-trivial changes.

---

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/). All participants are expected to uphold a respectful and inclusive environment. See `CODE_OF_CONDUCT.md` in the repository root for the full text.

---

## License

Contributions are licensed under the same terms as the project. By submitting a pull request, you agree that your contribution will be licensed under the project's open-source license.
