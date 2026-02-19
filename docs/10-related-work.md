# 10. Relationship to Existing Work

[< Security Analysis](09-security.md) | [Open Questions >](11-open-questions.md)

## 10.1 Related Concepts

- **Capability-based security (WASI):** WASI's capability model for WebAssembly is philosophically aligned — components get explicit, minimal capabilities rather than ambient authority. SHA applies this to the stochastic layer.
- **Microkernel architecture:** SHA's bus controller is analogous to a microkernel — a minimal trusted core that mediates all inter-component communication. The CUs and ALUs are like user-space servers.
- **Hardware TEEs (Trusted Execution Environments):** Intel SGX, ARM TrustZone — hardware-enforced isolation of sensitive computation. SHA's instruction memory isolation is analogous but applied to the inference layer.
- **RBAC/ABAC:** SHA's policy segment and bus controller implement access control, but at the architectural level rather than the application level.

## 10.2 Prior Art in AI Security

- Anthropic's system prompt / user message role distinction is a weak form of I-Bus / D-Bus separation.
- OpenAI's structured outputs enforce dispatch schema constraints (related to X-Bus protocol).
- Google's Gemini prompt caching separates "cached" (instruction-like) and "uncached" (data-like) context.
- Various "sandwich" defense techniques (instruction-data-instruction) attempt application-layer bus isolation.

SHA formalizes and strengthens these ad-hoc techniques into a coherent architectural framework.
