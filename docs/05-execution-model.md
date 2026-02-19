# 5. Scheduling and Execution Model

[< Bus Protocols and Enforcement](04-bus-protocols.md) | [Modified Harvard Considerations >](06-modified-harvard.md)

## 5.1 The Stochastic Cycle

A traditional CPU executes billions of instructions per second, each taking nanoseconds. A stochastic CU executes one "inference cycle" in seconds, but each cycle accomplishes what might require thousands of conventional instructions. This inverts the traditional scheduling model.

### 5.1.1 Cycle Anatomy

```mermaid
sequenceDiagram
    participant BC as Bus Controller
    participant CU as Control Unit
    participant ALU as ALU / I/O
    participant DMEM as Data Memory

    rect rgba(45, 125, 154, 0.15)
        Note over BC,DMEM: 1. FETCH
        BC->>BC: Compose context window from IMEM (I-Bus) + DMEM (D-Bus)
    end

    rect rgba(123, 45, 142, 0.15)
        Note over BC,CU: 2. DECODE
        BC->>CU: Deliver composed context
        CU->>CU: Process input (implicit)
    end

    rect rgba(123, 45, 142, 0.15)
        Note over CU: 3. EXECUTE
        CU->>CU: Produce output (reasoning, dispatch commands, responses)
    end

    rect rgba(90, 106, 191, 0.15)
        Note over CU,ALU: 4. DISPATCH
        CU->>DMEM: D-Bus writes (responses → DMEM-CONV, plans → IMEM-CTX)
        CU->>ALU: X-Bus dispatches (tool calls → ALU, sub-tasks → other CUs)
        CU->>ALU: I/O dispatches (messages → I/O-MSG, web requests → I/O-WEB)
    end

    rect rgba(45, 142, 94, 0.15)
        Note over ALU,DMEM: 5. WRITEBACK
        ALU->>DMEM: Results return via D-Bus
    end

    rect rgba(142, 107, 45, 0.15)
        Note over BC,DMEM: 6. REPEAT
        DMEM-->>BC: Next cycle begins with updated DMEM state
    end
```

### 5.1.2 Cycle Scheduling

Because CU cycles are expensive (seconds, dollars), scheduling matters more than in conventional systems:

**Eager dispatch:** When a CU identifies multiple independent operations, dispatch them simultaneously to ALU cores and other CUs. Don't serialize unnecessarily.

**Batched context:** Accumulate multiple inbound events (messages, tool results) before triggering a CU cycle, rather than cycling per-event.

**Tiered CU routing:** Route simple tasks to cheaper/faster CUs (smaller models) and complex tasks to more capable CUs. The bus controller or a lightweight router can make this determination without burning a full CU cycle.

**Speculative execution:** For high-latency operations, dispatch the likely next operation before the current one completes. If wrong, discard. The cost of a wasted ALU cycle is trivial compared to a wasted CU cycle.

## 5.2 Determinism and Verification

Because the CU is stochastic, the architecture must handle **nondeterministic execution** as a first-class concern.

**Verification strategies:**

- **Post-hoc validation:** After a CU produces a dispatch, the bus controller validates it against IMEM-POLICY before execution. Invalid dispatches are rejected deterministically.
- **Consensus:** For high-stakes operations, route the same input to multiple CUs (same or different models) and require agreement.
- **Rollback:** ALU operations are executed in sandboxed environments. If a CU cycle produces an unexpected result, the ALU state can be rolled back.
- **Confidence gating:** CU output includes confidence signals. Low-confidence dispatches are routed to human review via I/O rather than executed.
