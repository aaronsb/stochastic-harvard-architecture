# 9. Security Analysis

[< Implementation Strategies](08-implementation.md) | [Relationship to Existing Work >](10-related-work.md)

## 9.1 Threat Model

| Threat | Von Neumann (current) | SHA |
|---|---|---|
| **Prompt injection via user input** | Data enters instruction stream. Model may follow injected instructions. | Data stays on D-Bus. Cannot reach IMEM. CU may still be influenced but cannot be overridden. |
| **Prompt injection via tool output** | Tool results concatenated into context alongside instructions. | Tool results → DMEM-TOOL (D-Bus). Separate from instruction stream. |
| **Malicious skill/plugin** | Skill content mixed with user data in context. Hard to audit. | Skills installed via verification gate to IMEM-SKILL. Auditable. Isolated from D-Bus content. |
| **Data exfiltration** | Model can be tricked into sending sensitive data to external services. | Bus controller enforces I/O policy from IMEM-POLICY. Unauthorized outbound blocked deterministically. |
| **Instruction corruption** | Adversary modifies system prompt via data channel. | IMEM-SYS is write-locked. No D-Bus path to IMEM. |
| **CU misbehavior (stochastic fault)** | Model ignores instructions. No architectural backstop. | Bus controller validates all dispatches against IMEM-POLICY. Invalid operations rejected. |

## 9.2 Residual Risks

SHA does not eliminate all risks. Residual risks include:

- **CU influence:** The CU reads data and may be influenced by adversarial content to make poor decisions within its instruction boundaries. The architecture prevents data from *replacing* instructions but cannot prevent data from *affecting reasoning about* instructions. This is a model robustness problem, not an architecture problem. See [ADR-300](architecture/security/ADR-300-cu-influence-mitigation-strategies.md) for mitigation strategies including least-privilege scoping, output schema constraints, anomaly detection, and escalation gates.
- **IMEM-CTX poisoning:** Because IMEM-CTX is writable by CUs and derived from reasoning about untrusted data, a CU influenced by adversarial content could write a compromised execution plan that becomes part of the trusted instruction stream for subsequent cycles. See [ADR-200](architecture/bus/ADR-200-imem-ctx-write-semantics-and-trust-model.md) for the graduated trust model and CTX-specific verification gate design.
- **Middleware isolation gap:** At the middleware implementation tier, the bus controller composes the context window but the model's attention mechanism processes it as a single undifferentiated sequence. Structural injection is prevented but influence is not. See [ADR-100](architecture/core/ADR-100-middleware-bus-isolation-boundaries.md) for the three-tier isolation model.
- **Verification gate compromise:** If the verification gate is buggy or misconfigured, adversarial content could be promoted from D-Bus to IMEM. The gate must be small, auditable, and rigorously tested.
- **Bus controller compromise:** The bus controller is the trusted computing base. If compromised, all guarantees are void. It must be minimal, formally verifiable if possible, and not itself implemented using stochastic components.
- **Side channels:** CU behavior may leak instruction memory content via output patterns. This is analogous to speculative execution side channels in conventional hardware.
