# Stochastic Harvard Architecture (SHA)

## Draft Specification v0.1

**Status:** Exploratory draft — not affiliated with any organization
**Date:** February 2026
**Abstract:** This document describes an architectural framework for AI agent systems inspired by the Harvard computer architecture. It proposes a formal separation of instruction and data paths in stochastic computing systems to address fundamental security, reliability, and composability problems in current agent designs.

## Document Structure

| Section | Title | Summary |
|---------|-------|---------|
| [01](01-motivation.md) | Motivation | The von Neumann problem, the Harvard solution, why "stochastic" |
| [02](02-architecture.md) | Architectural Overview | Core components, bus architecture, system diagram |
| [03](03-components.md) | Component Specifications | CU, ALU, IMEM, DMEM, I/O channel specs |
| [04](04-bus-protocols.md) | Bus Protocols and Enforcement | Bus controller, context window composition, verification gates |
| [05](05-execution-model.md) | Scheduling and Execution Model | Stochastic cycle anatomy, scheduling, determinism |
| [06](06-modified-harvard.md) | Modified Harvard Considerations | Pure vs. modified, mapping to existing systems |
| [07](07-heterogeneous-cu.md) | Heterogeneous CU Composition | Multi-core analogy, dispatch routing, access control |
| [08](08-implementation.md) | Implementation Strategies | Middleware, API-level, model architecture, browser runtime |
| [09](09-security.md) | Security Analysis | Threat model, residual risks |
| [10](10-related-work.md) | Relationship to Existing Work | Capability security, microkernels, TEEs, prior art |
| [11](11-open-questions.md) | Open Questions | Seven unresolved design questions |
| [12](12-glossary.md) | Glossary | Term definitions |

The monolithic spec is maintained at [`stochastic-harvard-architecture-spec.md`](../stochastic-harvard-architecture-spec.md) in the repository root.

---

*This document is an exploratory draft. It describes a conceptual framework, not a finished specification. The architecture is proposed as a direction for research and discussion, not as a standard. Contributions, critiques, and experiments are welcome.*
