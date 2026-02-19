# Stochastic Harvard Architecture (SHA)

An architectural framework for AI agent systems that applies Harvard architecture principles — separate instruction and data buses — to address prompt injection structurally rather than behaviorally.

Current AI agent systems are von Neumann: instructions and data share one address space (the context window). Prompt injection is a structural consequence, not a bug. SHA separates them architecturally, below the model, enforced by deterministic middleware.

## Specification

The spec is in [`docs/`](docs/00-overview.md), organized as numbered sections:

- [00 Overview](docs/00-overview.md) — Document map
- [01 Motivation](docs/01-motivation.md) — Why this exists
- [02 Architecture](docs/02-architecture.md) — Components and buses
- [03 Components](docs/03-components.md) — CU, ALU, IMEM, DMEM, I/O
- [04 Bus Protocols](docs/04-bus-protocols.md) — Controller, context composition, verification gates
- [05 Execution Model](docs/05-execution-model.md) — Stochastic cycle, scheduling
- [06 Modified Harvard](docs/06-modified-harvard.md) — Pure vs. modified, existing systems
- [07 Heterogeneous CU](docs/07-heterogeneous-cu.md) — Multi-core composition
- [08 Implementation](docs/08-implementation.md) — Middleware through model-native
- [09 Security](docs/09-security.md) — Threat model, residual risks
- [10 Related Work](docs/10-related-work.md) — Prior art
- [11 Open Questions](docs/11-open-questions.md) — Unresolved design questions
- [12 Glossary](docs/12-glossary.md) — Terms

## Status

Exploratory draft (v0.1). Not affiliated with any organization. This is a conceptual framework proposed as a direction for research and discussion.

## Architecture Decisions

Design decisions are tracked as ADRs in [`docs/architecture/`](docs/architecture/).
