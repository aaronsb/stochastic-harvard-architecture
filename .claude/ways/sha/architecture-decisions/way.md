---
description: ADR workflow for SHA design decisions
trigger:
  keywords:
    - adr
    - decision
    - trade off
    - design choice
    - we should use
    - approach
session_once: true
---

# Architecture Decisions

Use ADRs for design decisions in this project. The ADR tool is at `docs/scripts/adr`.

## Domains

| Domain | Range | For |
|--------|-------|-----|
| core | 100-199 | Fundamental SHA components, bus architecture, memory model |
| bus | 200-299 | I-Bus, D-Bus, X-Bus protocol design, bus controller, verification gates |
| security | 300-399 | Threat model, isolation guarantees, trust boundaries |
| implementation | 400-499 | Middleware, API-level, model architecture strategies |

## When to Write an ADR

- Choosing a bus protocol format (MCP-based vs. custom)
- Defining verification gate semantics for IMEM-CTX
- Selecting an implementation strategy for the bus controller
- Making security/trust boundary decisions
- Any design choice with meaningful trade-offs

## Quick Start

```bash
docs/scripts/adr new bus "MCP as X-Bus Protocol"
docs/scripts/adr new security "IMEM-CTX Write Verification"
docs/scripts/adr list --group
```
