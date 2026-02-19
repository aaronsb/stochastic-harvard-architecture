---
description: SHA terminology consistency
trigger:
  keywords:
    - bus
    - IMEM
    - DMEM
    - CU
    - ALU
    - instruction memory
    - data memory
    - control unit
    - verification gate
    - bus controller
session_once: true
---

# SHA Terminology

Use consistent terminology from the [glossary](docs/12-glossary.md):

| Term | Correct Usage | Not |
|------|--------------|-----|
| **CU** | Control Unit (stochastic inference engine) | "agent", "model" when referring to architectural role |
| **ALU** | Arithmetic Logic Unit (deterministic compute) | "tool", "sandbox" when referring to architectural role |
| **IMEM** | Instruction Memory (with segments: SYS, SKILL, POLICY, CTX) | "system prompt", "config" |
| **DMEM** | Data Memory (with segments: FILE, CONV, TOOL, VEC, SCRATCH) | "context", "state" |
| **I-Bus** | Instruction Bus (IMEM → CU, isolated) | "system channel" |
| **D-Bus** | Data Bus (shared, bidirectional, untrusted) | "data channel" |
| **X-Bus** | Dispatch Bus (CU → CU, CU → ALU) | "command channel" |
| **Verification Gate** | Controlled D-Bus → IMEM crossing | "filter", "validator" |
| **Bus Controller** | Deterministic TCB mediating all bus traffic | "orchestrator", "router" |

When discussing existing systems (Claude Code, MCP, etc.), use their native terminology but map to SHA terms explicitly.
