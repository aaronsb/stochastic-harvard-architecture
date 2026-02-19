---
status: Draft
date: 2026-02-19
deciders:
  - aaronsb
related: [ADR-200, ADR-300]
---

# ADR-301: Bus Supervisor Model for Dispatch Verification

## Context

ADR-300 defines layered mitigations for CU influence attacks: least-privilege scoping, output schema constraints, anomaly detection, and escalation. ADR-200 defines a verification gate for IMEM-CTX writes with schema validation, scope checking, and diff-based change detection. Both rely on deterministic checks.

The gap: deterministic checks catch structurally invalid or out-of-scope dispatches, but cannot evaluate whether a structurally valid dispatch is *semantically consistent* with the CU's instructions. A compromised plan that is schema-valid, within scope, and proportionate to the input can still serve an adversary's goals.

Reinforcement learning offers a model for this: the **supervisor** (critic/reward model) evaluates the **policy's** (CU's) outputs against a learned representation of intended behavior. The supervisor doesn't need to be as capable as the policy — it needs to discriminate between aligned and misaligned outputs.

Applied to SHA, a bus supervisor is a stochastic component at the verification gate that evaluates CU dispatches against the instruction context. Unlike the bus controller (which is deterministic and must remain so), the supervisor is an inference engine that adds a probabilistic confidence signal to the deterministic validation pipeline.

## Decision

We define a **bus supervisor spectrum** — a range of supervisor implementations that trade off latency, cost, and detection capability. Implementations can choose their position on this spectrum based on the risk profile of the operations being supervised.

### The Supervisor Spectrum

```
Low latency / Low cost                              High latency / High capability
─────────────────────────────────────────────────────────────────────────────────────
Embedding     Classifier     Small LLM        Reasoning LLM       Consensus Panel
similarity    (fine-tuned)   (fast, scoped)   (full comprehension) (multi-model)
```

#### Level 0: Embedding Similarity (microseconds)

Continuous embedding match between the instruction context (IMEM) and the proposed dispatch/plan output.

- **How**: Embed the current IMEM-CTX goal + IMEM-POLICY constraints into a vector. Embed the proposed dispatch. Compute cosine similarity. Flag if below threshold.
- **Detects**: Dispatches that are semantically distant from the stated goal. An email summarization CU dispatching a file deletion scores low similarity to "summarize recent messages."
- **Misses**: Dispatches that are semantically adjacent but subtly misaligned. "Forward email to user's archive" vs. "forward email to attacker's address" may embed similarly.
- **Cost**: Negligible. Embedding models run in milliseconds. Can run on every dispatch without meaningful overhead.
- **Trust model**: Very low confidence signal. Useful as a fast pre-filter, not as a gatekeeper.

#### Level 1: Fine-Tuned Classifier (milliseconds)

A small model trained specifically to classify dispatches as aligned/misaligned given an instruction context.

- **How**: Train a classifier on (instruction_context, dispatch) pairs labeled as aligned or misaligned. Input is the IMEM instruction segments + proposed dispatch. Output is a confidence score.
- **Detects**: Dispatch patterns that match known attack signatures. Can learn to flag "the instruction says read-only but the dispatch writes" or "the instruction mentions no external recipients but the dispatch targets an external I/O."
- **Misses**: Novel attack patterns outside the training distribution. The classifier is only as good as its training data.
- **Cost**: Low. Small classifiers run in single-digit milliseconds. Suitable for every dispatch.
- **Trust model**: Medium confidence for known patterns. Low confidence for novel situations.

#### Level 2: Small Scoped LLM (100ms–2s)

A small, fast LLM (e.g., Haiku-class) given a narrow prompt: "Given these instructions and this proposed action, is the action consistent with the instructions? Answer yes/no with brief reasoning."

- **How**: Compose a minimal context from IMEM-SYS + IMEM-CTX goal + the proposed dispatch. No D-Bus content — the supervisor reads only instructions and the proposed action, not the data that influenced the CU. This is critical: the supervisor must not be exposed to the same adversarial data that influenced the primary CU.
- **Detects**: Semantic misalignment between instructions and dispatches. Can reason about intent, not just pattern-match.
- **Misses**: Subtle influence where the dispatch is genuinely ambiguous given the instruction context. The supervisor's stochastic nature means it can also be wrong.
- **Cost**: Moderate. Suitable for dispatches above a risk threshold (I/O operations, IMEM-CTX writes, destructive actions).
- **Trust model**: Medium-high confidence. The I-Bus-only input isolates the supervisor from the same influence vector that may have compromised the primary CU.

#### Level 3: Reasoning LLM (2–30s)

A full-capability LLM with extended thinking, given the complete instruction context and asked to evaluate the proposed plan or dispatch in detail.

- **How**: Full IMEM context (SYS + POLICY + SKILL + CTX) plus the proposed dispatch/plan. The supervisor performs multi-step reasoning about whether the proposed actions serve the stated goals and comply with the stated policies.
- **Detects**: Complex misalignment, subtle policy violations, multi-step attack plans where individual steps look benign but the sequence is malicious.
- **Misses**: Adversarial inputs specifically crafted to fool both the primary CU and the supervisor. If the attacker knows the supervisor's model and prompt, they can craft bypasses.
- **Cost**: High. Latency comparable to the primary CU cycle. Suitable only for high-stakes operations: IMEM-CTX plan rewrites, irreversible I/O, security-sensitive dispatches.
- **Trust model**: High confidence but not infallible. The supervisor is itself stochastic.

#### Level 4: Consensus Panel (30s+)

Multiple independent supervisor models evaluate the same dispatch. Agreement required for approval.

- **How**: Route the proposed dispatch to N independent models (different providers, different architectures). Require M-of-N agreement.
- **Detects**: Attacks that exploit model-specific vulnerabilities. If an influence attack works on Claude but not GPT, the panel catches it.
- **Misses**: Universal model vulnerabilities (attacks that fool all models). These are rare but theoretically possible.
- **Cost**: Very high. N × supervisor cost. Suitable only for the most critical operations.
- **Trust model**: Highest available confidence.

### Supervisor Routing Policy

The bus controller selects the supervisor level based on operation risk, defined in IMEM-POLICY:

| Operation Type | Default Supervisor Level |
|---|---|
| DMEM read/write | Level 0 (embedding) or none |
| Routine ALU dispatch | Level 0 (embedding) |
| I/O read operations | Level 1 (classifier) |
| I/O write operations | Level 2 (small LLM) |
| IMEM-CTX plan update | Level 2 (small LLM) |
| Irreversible I/O (send, delete, publish) | Level 3 (reasoning LLM) |
| IMEM-POLICY or IMEM-SKILL modification | Level 3 (reasoning LLM) + human escalation |

These defaults are configurable. High-security deployments can raise all levels. Cost-sensitive deployments can lower them, accepting more risk.

### Critical Design Constraint: Supervisor Isolation

The supervisor must **not** read D-Bus content. If the supervisor sees the same adversarial data that influenced the primary CU, it may be influenced in the same way. The supervisor's input is:

- IMEM segments (instructions — trusted)
- The proposed dispatch or plan (CU output — being evaluated)
- Optionally, the dispatch history (for anomaly context)

**Not included**: DMEM-CONV, DMEM-TOOL, DMEM-FILE, or any other D-Bus content. The supervisor evaluates "is this action consistent with the instructions?" without seeing the data that motivated the action. This is the key architectural property that makes the supervisor resistant to the same influence vector.

### Information-Theoretic Limits

The supervisor faces a fundamental detection limit: if a compromised dispatch is indistinguishable from a legitimate dispatch given only the instruction context (no data context), the supervisor cannot detect it. This happens when:

- The adversary's desired action is within the CU's normal behavioral range
- The dispatch parameters don't violate any policy or schema constraint
- The action would be reasonable given *some* possible data input

This is irreducible. The supervisor reduces the attack surface to cases where the adversary can craft dispatches that are both harmful and indistinguishable from legitimate behavior — a much harder bar than "craft text that confuses the model."

## Consequences

### Positive

- Provides a probabilistic defense layer for the semantic gap that deterministic checks cannot cover
- The spectrum allows cost/latency/security tradeoff tuning per deployment
- Supervisor isolation from D-Bus content makes it resistant to the same influence attacks that compromise the primary CU
- Compatible with all three implementation tiers from ADR-100

### Negative

- Adds stochastic components to the verification pipeline, which the spec previously positioned as purely deterministic
- Supervisor models need training/selection, adding operational complexity
- Latency overhead for Level 2+ supervisors impacts cycle time
- The supervisor is itself fallible — it adds a defense layer, not a guarantee

### Neutral

- The bus controller remains deterministic and retains its role as the TCB. The supervisor provides an advisory signal to the deterministic gate, not a replacement for it
- This mirrors the RL pattern: the policy proposes, the critic evaluates, the environment (bus controller) enforces hard constraints

## Alternatives Considered

- **Deterministic checks only**: The current approach in ADR-200 and ADR-300. Sufficient for structural validation but cannot evaluate semantic consistency. The supervisor complements, not replaces, deterministic checks.
- **Single fixed supervisor level**: Rejected — one level cannot serve all risk profiles. High-security operations need reasoning-level supervision while routine operations can't afford the latency.
- **Supervisor reads D-Bus content for full context**: Rejected — exposing the supervisor to the same adversarial data that influenced the CU defeats the purpose. The supervisor's value comes from its isolated perspective.
