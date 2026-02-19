---
status: Draft
date: 2026-02-19
deciders:
  - aaronsb
related: [ADR-200, ADR-300]
---

# ADR-100: Middleware Bus Isolation Boundaries

## Context

The SHA specification proposes a middleware implementation (§8.1) where a deterministic bus controller composes the context window from separated I-Bus and D-Bus sources before calling an LLM API. A valid critique observes that this is partially illusory: regardless of how carefully the middleware separates instruction and data content, the model's self-attention mechanism processes the entire context window as a single undifferentiated token sequence. The model has no architectural mechanism to respect the bus boundary.

This raises the question: what security guarantees does the middleware layer *actually* provide, and where exactly does its protection end?

## Decision

We explicitly define three tiers of bus isolation with clear boundaries on what each tier guarantees and where it fails:

### Tier 1: Composition Isolation (Middleware — available today)

The bus controller owns context window composition. This prevents:

- **Structural injection**: Untrusted data cannot be placed in the system prompt region of the API call. Even if the model's attention doesn't differentiate roles, API providers do process system/user/assistant roles through different code paths. Content in the system role has measurably different behavioral weight.
- **Cross-contamination at assembly time**: Tool outputs cannot overwrite skill definitions. User messages cannot modify policies. A poisoned web page cannot masquerade as a system instruction at the API request level. The bus controller's composition logic is deterministic and cannot be manipulated by content in either region.
- **Uncontrolled context growth**: The bus controller enforces capacity budgets per memory segment, preventing an adversary from flooding the data bus to crowd out instruction content.

What Tier 1 does **not** prevent: The model may still be *influenced* by adversarial data content to deviate from its instructions within the composed context. The separation is at the API boundary, not the attention boundary.

### Tier 2: API-Level Isolation (Provider support — medium-term)

LLM providers add explicit guarantees about how instruction-region and data-region tokens are processed. Options include differential attention weighting, separate processing pathways, or contractual guarantees about role handling. This narrows the influence gap but doesn't eliminate it.

### Tier 3: Model-Native Isolation (Architecture changes — long-term)

Dual-stream attention, typed token embeddings, or hardware-enforced context regions. This is the only tier that can close the gap between "data cannot replace instructions" and "data cannot influence reasoning about instructions."

### The honest claim

SHA middleware (Tier 1) provides **structural guarantees against the most common prompt injection vectors**: system prompt hijacking, skill installation via data channels, and policy modification through untrusted input. It does not provide guarantees against *influence attacks* where the model is manipulated within its allowed behavioral boundaries. The spec must state this boundary clearly rather than implying stronger guarantees than the middleware can deliver.

## Consequences

### Positive

- Clear expectation-setting prevents the "middleware illusion" critique — we acknowledge the limitation up front
- Tier 1 is still genuinely valuable: it eliminates an entire class of structural injection attacks without requiring any model changes
- The tiered roadmap gives providers and implementers a clear path from "useful today" to "architecturally sound"

### Negative

- Acknowledging Tier 1 limitations may reduce perceived value of the middleware approach
- The influence gap means the bus controller cannot provide *complete* protection against prompt injection, only structural protection

### Neutral

- This motivates ADR-300 (CU influence mitigation) as the complementary strategy for the influence gap
- Provider adoption of Tier 2 becomes a critical dependency for stronger guarantees

## Alternatives Considered

- **Claim middleware provides complete isolation**: Rejected — this is dishonest and would be quickly disproven. The spec's credibility depends on honest treatment of limitations.
- **Dismiss middleware as useless without model support**: Rejected — Tier 1 eliminates real attack vectors (system prompt injection via tool output, skill hijacking via data channels) that cause real harm today. The perfect shouldn't be the enemy of the good.
- **Wait for Tier 3 before specifying the architecture**: Rejected — the architectural framework is valuable independent of implementation tier. Designing the bus separation now means implementations can progressively strengthen isolation as support becomes available.
