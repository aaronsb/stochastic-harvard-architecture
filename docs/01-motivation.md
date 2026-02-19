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
