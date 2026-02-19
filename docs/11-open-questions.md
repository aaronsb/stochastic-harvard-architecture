# 11. Open Questions

[< Relationship to Existing Work](10-related-work.md) | [Glossary >](12-glossary.md)

1. **IMEM-CTX write semantics:** The execution context segment is writable by CUs during operation. How do we prevent a CU that has been influenced by adversarial data from writing compromised plans to IMEM-CTX? Options include: validation of plan content by the bus controller, requiring human approval for plan changes, or treating IMEM-CTX as a "gray zone" with reduced trust.

2. **Context window limits:** Current models have finite context windows. How do we prioritize IMEM vs. DMEM content when the combined instruction + data exceeds capacity? Who makes the eviction decision? If a CU makes it, the eviction policy is stochastic. If the bus controller makes it, it's deterministic but less intelligent.

3. **Model API support:** Current APIs (Anthropic, OpenAI, Google) provide role-based message structure but no guarantee that the model actually treats roles differently at the attention level. True SHA requires either API-level enforcement or model architecture changes. What's the minimum API change needed?

4. **Performance overhead:** Bus controller mediation adds latency to every cycle. Context window composition from multiple sources adds complexity. Is the overhead acceptable? How does it compare to the overhead of current prompt engineering defenses?

5. **Observability:** How do we provide meaningful visibility into bus traffic, IMEM state, and CU decision-making without overwhelming operators? What does a "debugger" look like for a stochastic Harvard machine?

6. **Multi-tenant isolation:** If multiple users share a system (e.g., a shared OpenClaw instance), how do we partition IMEM and DMEM between tenants? Is per-tenant bus isolation needed, or is per-tenant access control within shared buses sufficient?

7. **Emergent instruction sets:** The "instruction set" of an LLM CU is not fixed — it's whatever the model can do, which changes with each model generation. How do we maintain architectural contracts when the underlying CU capabilities shift? How do we version and test the system's capabilities?
