---
description: Guidance for editing SHA specification documents
trigger:
  file_patterns:
    - "docs/*.md"
session_once: true
---

# Editing SHA Specification

## Document Structure

The spec lives in `docs/00-12` as numbered sections. Each section is self-contained with prev/next navigation links.

| Section | Content |
|---------|---------|
| 00 | Overview and document map |
| 01 | Motivation (von Neumann problem, Harvard solution) |
| 02 | Architectural overview (components, buses, system diagram) |
| 03 | Component specs (CU, ALU, IMEM, DMEM, I/O) |
| 04 | Bus protocols (controller, context composition, verification gates) |
| 05 | Execution model (stochastic cycle, scheduling) |
| 06 | Modified Harvard (pure vs. modified, existing system mapping) |
| 07 | Heterogeneous CU composition (multi-core, dispatch routing) |
| 08 | Implementation strategies (middleware through model-native) |
| 09 | Security analysis (threat model, residual risks) |
| 10 | Related work |
| 11 | Open questions |
| 12 | Glossary |

## Diagram Standards

Use Mermaid with the project color palette:
- `#2D7D9A` teal — IMEM / instruction-related
- `#7B2D8E` purple — CU / control
- `#2D8E5E` green — DMEM / data
- `#C2572A` burnt orange — I/O / external
- `#5A6ABF` slate blue — ALU / compute
- `#8E6B2D` amber — boundaries / config
- `#FFFFFF` white text on colored fills

Choose diagram type by content (see docs way for mapping).

## Cross-References

Use relative links between sections: `[§4.3](04-bus-protocols.md#43-verification-gates)`

When adding new content, determine which existing section it belongs to. Create a new numbered section only if the content doesn't fit any existing one.
