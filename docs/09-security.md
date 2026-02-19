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

- **CU influence:** The CU reads data and may be influenced by adversarial content to make poor decisions within its instruction boundaries. The architecture prevents data from *replacing* instructions but cannot prevent data from *affecting reasoning about* instructions. This is a model robustness problem, not an architecture problem.
- **Verification gate compromise:** If the verification gate is buggy or misconfigured, adversarial content could be promoted from D-Bus to IMEM. The gate must be small, auditable, and rigorously tested.
- **Bus controller compromise:** The bus controller is the trusted computing base. If compromised, all guarantees are void. It must be minimal, formally verifiable if possible, and not itself implemented using stochastic components.
- **Side channels:** CU behavior may leak instruction memory content via output patterns. This is analogous to speculative execution side channels in conventional hardware.
