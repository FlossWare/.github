# FlossWare

**Free-first, modular infrastructure and AI-assisted engineering.**

FlossWare builds reusable engineering foundations and reference implementations using open standards, explicit configuration, and loosely coupled architectures.

## Featured Projects

| Repository | Purpose |
|---|---|
| [standards](https://github.com/FlossWare/standards) | Authoritative FlossWare-wide standards and architecture decisions |
| [commons-java](https://github.com/FlossWare/commons-java) | Shared Java foundation libraries |
| [tftp-os](https://github.com/FlossWare/tftp-os) | Reference implementation for infrastructure provisioning and automation |

## Architecture Principles

FlossWare systems follow these core ideas:

- **Configuration is the source of truth**
- **Minimal behavior by default; capabilities are explicitly enabled**
- **Components are modular and composable**
- **Open standards over vendor lock-in**
- **Loose coupling through stable contracts**

## Reference Architecture

```text
                 Clients / Agents
                       |
          REST APIs / MCP Tool Interfaces
                       |
              Service Boundaries
                       |
        +--------------+--------------+
        |                             |
  Stored Procedures              Message Bus
        |                             |
        v                             v
    Databases                 Event Consumers
```

## Integration Model

- **REST** provides synchronous service contracts.
- **MCP** provides AI agent and tool integration contracts.
- **Message buses** provide asynchronous workflows and event-driven integration.
- **Stored procedures** provide database abstraction where they add value.
- **ADRs** preserve architectural decisions.

## Documentation

Authoritative normative standards and architecture decisions live in:

- [FlossWare/standards](https://github.com/FlossWare/standards) — normative rules and architecture standards
- [FlossWare/axium](https://github.com/FlossWare/axium) — canonical non-code knowledge corpus

See the repository ADRs for detailed architectural decisions.

## Contributing

Contributions should align with the FlossWare standards. Projects should favor clarity, composability, and maintainable open designs over unnecessary complexity.
