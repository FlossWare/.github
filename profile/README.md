# FlossWare

**Composable software capabilities, contract-first architecture, and AI-assisted engineering.**

FlossWare develops reusable software contracts, implementations, and engineering foundations. Its architecture emphasizes explicit boundaries, language-neutral contracts, composable capabilities, and evidence-backed decisions.

## Start here

| Repository | Responsibility |
|---|---|
| [standards](https://github.com/FlossWare/standards) | Authoritative normative rules and cross-project standards |
| [knowledge](https://github.com/FlossWare/knowledge) | Canonical Git-backed corpus for durable knowledge, decisions, research outcomes, evidence, and provenance |
| [research](https://github.com/FlossWare/research) | Investigations, experiments, and architectural exploration before conclusions are promoted |
| [loom](https://github.com/FlossWare/loom) | Language-neutral Loom contracts and semantic model |
| [loom-python](https://github.com/FlossWare/loom-python) | Python implementation of the Loom contract |
| [loom-ai](https://github.com/FlossWare/loom-ai) | AI-domain contracts and semantics built on Loom |
| [loom-ai-python](https://github.com/FlossWare/loom-ai-python) | Python implementation of the AI-domain contracts |

## Architecture

Loom is a language-neutral execution substrate. AI is a domain built on Loom, not the definition of Loom itself. Capability repositories provide focused functionality behind explicit contracts; clients integrate through supported interfaces rather than becoming part of Loom's internal implementation.

```text
Foundational contracts
        |
        +-- language implementations
        |
        +-- domain contracts
                 |
                 +-- domain implementations

Knowledge records durable context and provenance.
Research investigates questions and tests hypotheses.
Standards defines normative rules.
Software repositories own executable contracts, tests, and implementations.
```

## Where things belong

- **Standards** prescribe cross-project rules.
- **Knowledge** preserves durable, reusable knowledge and its provenance.
- **Research** records investigations and experiments that may still be incomplete or wrong.
- **Software repositories** own their contracts, executable implementations, tests, and implementation-specific documentation.
- **This GitHub organization profile** is an entry point, not a competing documentation or knowledge source.

The public documentation site is published from [FlossWare/FlossWare](https://github.com/FlossWare/FlossWare). It presents selected documentation; it does not replace the canonical knowledge corpus or software repositories.

## Engineering principles

- Keep foundational contracts language-neutral.
- Separate contracts from language-specific implementations.
- Keep domain semantics distinct from foundational contracts.
- Preserve provenance across research, decisions, derivation, and execution.
- Prefer explicit composition and stable boundaries over unnecessary coupling.
- Treat research conclusions as evidence to evaluate, not automatically as normative rules.

