# 1. Motivation

[< Overview](00-overview.md) | [Architectural Overview >](02-architecture.md)

## 1.1 The Von Neumann Problem in Current Agent Systems

Contemporary AI agent architectures (Claude Code, OpenClaw, Cursor, Devin, etc.) share a structural property with von Neumann computers: **instructions and data occupy the same address space.** In a von Neumann CPU, this means data can be executed as code (buffer overflows, code injection). In an LLM-based agent, this means data can be interpreted as instructions (prompt injection).

This is not a bug in any particular implementation. It is a **structural property of the architecture.** The context window of a large language model is a single, undifferentiated sequence of tokens. System prompts, user messages, tool outputs, file contents, and web page text are all concatenated into one stream. The model has no architectural mechanism to distinguish between "tokens I should follow as instructions" and "tokens I should process as data."

Every known defense against prompt injection operates at the application layer — instructing the model, via more tokens in the same stream, to please not treat certain other tokens as instructions. This is equivalent to preventing buffer overflows by writing careful C code. It works probabilistically. It fails structurally.

## 1.2 The Harvard Solution

The Harvard architecture, developed for the Harvard Mark I in 1944, solved an analogous problem in hardware by physically separating instruction memory and data memory with independent buses. The CPU reads instructions from one memory space and operates on data in another. Data cannot be executed as code because the two exist in different address spaces accessed through different physical pathways.

This document proposes applying the same structural separation to AI agent systems. Rather than relying on the model's ability to distinguish instruction tokens from data tokens within a shared context, we separate them architecturally — at the system level, below the model, enforced by the runtime rather than by the model's compliance.

## 1.3 Why "Stochastic"

Traditional Harvard architecture assumes a deterministic processor. The control unit fetches an instruction and executes it identically every time. This specification describes a system where the primary execution unit (the inference engine) is **fundamentally stochastic** — the same inputs may produce different outputs across invocations. This has cascading implications for scheduling, verification, error handling, and system design that differ from both traditional computing and traditional Harvard architecture.

## 1.4 What This Architecture Is and Is Not

**The LLM is the CPU.** This architecture treats the inference engine as a fixed component — a processor with known characteristics, accessed through a defined interface (the API), whose internal operation is not under our control. We do not propose changes to how models work. The inference is what it is: a stochastic function from input tokens to output tokens, delivered by remote APIs or local runtimes, with the characteristics and limitations of the current generation of language models.

**The entire purpose of this architecture is to manage execution around that fixed component.** SHA defines how instructions and data are organized, separated, and composed *before* they reach the model, and how the model's outputs are validated, routed, and constrained *after* they leave. It is a system architecture for the surrounding infrastructure — buses, memory, controllers, verification gates — not a proposal for how the model itself should change.

This is an honest scope boundary:

- SHA **does** define how to prevent untrusted data from being composed into the instruction stream.
- SHA **does** define how to validate CU outputs against policy before execution.
- SHA **does** define how to isolate execution environments and enforce least-privilege access.
- SHA **does not** claim to make the CU more reliable, more aligned, or less susceptible to influence.
- SHA **does not** require model architecture changes (though it can benefit from them — see [§8.2](08-implementation.md), [§8.3](08-implementation.md)).
- SHA **does not** eliminate the fundamental stochasticity of inference. It manages it.

The analogy to hardware is precise: CPU architects do not redesign the transistor. They change the arrangement of transistors — buses, caches, branch predictors, memory controllers — to get reliable, secure, observable computation from an imperfect physical substrate. Same transistor, different topology, different system-level guarantees. SHA does the same for an imperfect stochastic substrate.
