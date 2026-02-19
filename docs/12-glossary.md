# 12. Glossary

[< Open Questions](11-open-questions.md) | [Overview](00-overview.md)

| Term | Definition |
|---|---|
| **ALU** | Arithmetic Logic Unit — deterministic compute resource (container, sandbox, conventional CPU) |
| **Bus Controller** | Deterministic middleware component that mediates all inter-component communication and enforces isolation policies |
| **CU** | Control Unit — stochastic inference engine (LLM, diffusion model, or other generative model) |
| **D-Bus** | Data Bus — shared communication path for untrusted/general-purpose content |
| **DMEM** | Data Memory — storage for files, conversations, tool outputs, embeddings |
| **I-Bus** | Instruction Bus — protected communication path for trusted instructions and policies |
| **IMEM** | Instruction Memory — protected storage for system prompts, skills, policies, execution context |
| **SHA** | Stochastic Harvard Architecture |
| **Verification Gate** | Controlled pathway allowing content to cross from D-Bus to IMEM with validation |
| **X-Bus** | Dispatch Bus — command channel between CUs and between CUs and ALUs |
