# FlossWare Architecture Guide

> **Status: Historical / retired.** This document describes the pre-Loom FlossWare orchestration architecture and is retained only as repository history. It is **not** the current organization-wide architecture.

## Current architecture

The current FlossWare architecture is documented in the public FlossWare documentation repository:

- [FlossWare architecture](https://github.com/FlossWare/FlossWare)
- [Loom architecture](https://github.com/FlossWare/FlossWare/blob/main/docs/architecture/loom.md)
- [Orchestration layer](https://github.com/FlossWare/FlossWare/blob/main/docs/architecture/orchestration.md)
- [Model routing](https://github.com/FlossWare/FlossWare/blob/main/docs/architecture/routing.md)
- [Repository map](https://github.com/FlossWare/FlossWare/blob/main/docs/repositories.md)
- [Design philosophy](https://github.com/FlossWare/FlossWare/blob/main/docs/philosophy.md)
- [Engineering standards and ADRs](https://github.com/FlossWare/engineering-standards)

## Why this document was retired

The architecture formerly described a fixed physical fleet, SSH-dispatched workers, a centralized `aio-01` controller, and a large API-provider model pool as if those were universal architectural requirements. Those details may describe historical deployments, but they are not the current FlossWare architecture.

The current model is contract-first and provider-neutral. Loom provides the orchestration substrate, while inference, storage, search, graph, queue, tool, resource, and execution implementations remain replaceable. Local and cloud inference are both valid implementation choices.

Historical details such as machine names, IP addresses, hardware defects, SSH keys, Ansible role layouts, provider-specific model pools, and old fleet operational procedures should not be copied into new architecture documentation.

## Migration rule

When maintaining organization documentation, use the FlossWare repository and engineering standards as the sources of truth. If historical fleet material is needed for operational archaeology, consult Git history rather than treating this file as a living specification.
