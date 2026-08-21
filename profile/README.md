# FlossWare

**Open-source engineering foundations, AI orchestration, and modular systems.**

FlossWare builds reusable engineering foundations and reference implementations using open standards, explicit contracts, and replaceable components.

## Featured Projects

| Repository | Purpose |
|---|---|
| [loom-ai](https://github.com/FlossWare/loom-ai) | Provider-neutral AI orchestration substrate with pluggable contracts and backends |
| [engineering-standards](https://github.com/FlossWare/engineering-standards) | Architecture decisions, engineering principles, and development standards |
| [commons-java](https://github.com/FlossWare/commons-java) | Shared Java foundation libraries |

## Architecture

FlossWare favors **contracts over coupling** and **replaceability over centralized assumptions**.

```text
                    Applications / Agents
                             |
               SDK / CLI / REST / MCP
                             |
                    Loom orchestration
                             |
                    Contracts / Protocols
                             |
          +------------------+------------------+
          |                  |                  |
        Models             Data           Capabilities
     local / cloud     storage / search     tools / resources
          |                  |                  |
          +------------ replaceable -----------+
```

Loom is the primary AI orchestration substrate. Model providers, storage engines, queues, search systems, graph stores, tools, and other integrations are implementations behind contracts rather than universal architectural dependencies.

Local and cloud inference are both valid deployment choices. The appropriate implementation depends on requirements such as privacy, latency, cost, availability, model capability, and operational complexity.

## Engineering principles

- **Stable contracts over implementation coupling**
- **Provider independence and replaceability**
- **Minimal core with optional capabilities**
- **Open standards and explicit integration boundaries**
- **Verification separate from generation where appropriate**
- **ADRs preserve architectural decisions**
- **Dogfooding validates the architecture in real engineering workflows**

## Documentation

The canonical current architecture documentation lives in:

- [FlossWare organization documentation](https://github.com/FlossWare/FlossWare)
- [Loom architecture](https://github.com/FlossWare/FlossWare/blob/main/docs/architecture/loom.md)
- [Repository map](https://github.com/FlossWare/FlossWare/blob/main/docs/repositories.md)
- [Engineering standards](https://github.com/FlossWare/engineering-standards)

Historical fleet and orchestration material previously published under `.github/docs/` is being retired. It should not be interpreted as the current organization-wide architecture.

## Contributing

Contributions should align with the FlossWare engineering standards. Projects should favor clarity, composability, explicit contracts, and maintainable open designs over unnecessary complexity.
