---
status: Draft
date: 2026-02-19
deciders:
  - aaronsb
related: [ADR-100, ADR-300]
---

# ADR-200: IMEM-CTX Write Semantics and Trust Model

## Context

IMEM-CTX (Execution Context Segment) is the most problematic element of the SHA memory model. It stores the CU's current execution plan — task decomposition, active goals, reasoning chain state — and it is writable by CUs during operation.

This creates a structural tension: IMEM-CTX lives on the instruction bus and is read by CUs as part of their instruction stream, but its content is *derived from reasoning about untrusted data* on the D-Bus. A CU processing adversarial conversation history (DMEM-CONV) could formulate a malicious plan and write it to IMEM-CTX, where it becomes part of the trusted instruction stream for the next cycle.

This is the most important open design question in the spec (§11, Question 1). It is the gray zone between instruction and data — the place where the Harvard separation gets fuzzy.

## Decision

IMEM-CTX should be treated as a **distinct trust tier** — neither fully trusted (like IMEM-SYS) nor fully untrusted (like DMEM-CONV). We define a graduated trust model and a verification gate specific to IMEM-CTX writes.

### Trust Tiers in IMEM

| Segment | Trust Level | Write Path | Verification |
|---------|------------|------------|--------------|
| IMEM-SYS | Immutable during execution | Admin process only | Cryptographic authorization, hash-verified at boot |
| IMEM-POLICY | High trust, admin-writable | Admin process, policy update gate | Authorization + schema validation |
| IMEM-SKILL | High trust, gate-writable | Skill installation gate | Provenance + signature + authorization |
| **IMEM-CTX** | **Conditional trust, CU-writable** | **CU via CTX verification gate** | **Schema + scope + escalation (see below)** |

### IMEM-CTX Verification Gate

Every CU write to IMEM-CTX passes through a deterministic verification gate that enforces:

**1. Schema validation (deterministic, fast)**

Execution plans must conform to a strict schema:

```yaml
plan:
  goal: string           # What the CU is trying to accomplish
  steps:                 # Ordered list of planned operations
    - action: enum       # From a closed set of allowed action types
      target: ref        # Reference to a defined resource (ALU, I/O, DMEM segment)
      params: object     # Action-specific parameters
      justification: string  # Why this step is needed (for audit)
  constraints:           # Inherited from IMEM-POLICY
    - policy_ref: string
```

Plans that don't parse as valid schema are rejected. This prevents arbitrary text from being written to IMEM-CTX — the plan must be structurally well-formed.

**2. Scope validation (deterministic, medium)**

Each step in the plan is checked against the CU's access control matrix (§7.3) and IMEM-POLICY:

- Does the CU have permission to dispatch to the specified target?
- Are the parameters within allowed ranges?
- Does the plan reference only memory segments and I/O channels the CU is authorized for?

Steps that exceed the CU's authorized scope are rejected.

**3. Diff-based change detection (deterministic, fast)**

The gate compares the proposed plan against the previous IMEM-CTX state:

- What changed? Which steps were added, removed, or modified?
- Are the changes proportional to the new information received? (A small user message shouldn't cause a complete plan rewrite.)
- Do the changes introduce new capabilities or targets not present in the previous plan?

Disproportionate changes are flagged but not necessarily rejected — they're routed to escalation.

**4. Escalation threshold (configurable)**

Plan modifications that exceed a configurable risk threshold are routed to human review via I/O rather than auto-committed to IMEM-CTX:

- Plan introduces a new I/O target not in the original dispatch
- Plan adds steps with destructive operations (delete, overwrite, send)
- Plan modification rate exceeds threshold (multiple rewrites per cycle)
- Cumulative plan drift from original dispatch exceeds threshold

### What this doesn't solve

The IMEM-CTX gate validates *structure* and *scope* but not *intent*. A plan that is schema-valid, within scope, and proportionate to the input can still be subtly wrong in ways that serve an adversary's goals. Example: a plan that correctly summarizes email but selectively omits information that would alert the user to a problem.

This is the same residual risk identified in ADR-300 — it's a model quality problem that the architecture can constrain but not eliminate.

### The philosophical position

IMEM-CTX is not instruction memory in the same sense as IMEM-SYS. It is better understood as a **formalized scratchpad with instruction-like privileges**: the CU writes its working plan there, and the plan influences the next cycle's behavior, but the plan is constrained by the immutable layers above it (SYS, POLICY, SKILL). IMEM-CTX is the layer where the stochastic and deterministic parts of the system negotiate — the CU proposes, the gate validates, the policy constrains.

## Consequences

### Positive

- Schema validation eliminates arbitrary text injection into IMEM-CTX
- Scope validation enforces least privilege at the plan level
- Diff-based detection catches disproportionate plan changes that may indicate influence
- Escalation gives humans oversight of high-risk plan modifications
- The trust tier model makes the IMEM-CTX gray zone explicit rather than sweeping it under the rug

### Negative

- Schema validation requires defining and maintaining a plan schema, which may lag behind CU capabilities
- Diff-based detection needs tuning — too sensitive creates friction, too loose misses attacks
- Escalation gates add latency to plan modifications
- The plan schema constrains CU expressiveness — plans must fit the schema, which may not capture all valid planning patterns

### Neutral

- This positions IMEM-CTX as a hybrid space rather than forcing it into a pure instruction or pure data classification — honest about the architectural gray zone
- The gate design is compatible with all three implementation tiers from ADR-100

## Alternatives Considered

- **Treat IMEM-CTX as fully trusted**: Rejected — this ignores the influence vulnerability. If a CU writes a compromised plan to IMEM-CTX, it becomes trusted instruction content for subsequent cycles. Unacceptable.
- **Treat IMEM-CTX as untrusted data (move to DMEM)**: Rejected — execution plans need to persist across cycles and influence CU behavior in a way that data does not. Moving plans to DMEM would eliminate the useful property of plans-as-instructions and reduce the system to von Neumann for planning purposes.
- **Require human approval for all IMEM-CTX writes**: Rejected — too much friction. Most plan updates are routine (adding a step, marking completion) and don't warrant human review. Escalation should be risk-based, not universal.
- **Use a separate validation CU to check plans**: Considered but deferred — running a second inference call to validate the first is expensive and introduces recursion (who validates the validator?). This may be appropriate for Tier 2/3 implementations but is impractical for Tier 1 middleware. Schema and scope validation are deterministic and sufficient for the common case.
