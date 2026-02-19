# Stochastic Harvard Architecture (SHA)

## Draft Specification v0.1

**Status:** Exploratory draft — not affiliated with any organization  
**Date:** February 2026  
**Abstract:** This document describes an architectural framework for AI agent systems inspired by the Harvard computer architecture. It proposes a formal separation of instruction and data paths in stochastic computing systems to address fundamental security, reliability, and composability problems in current agent designs.

---

## 1. Motivation

### 1.1 The Von Neumann Problem in Current Agent Systems

Contemporary AI agent architectures (Claude Code, OpenClaw, Cursor, Devin, etc.) share a structural property with von Neumann computers: **instructions and data occupy the same address space.** In a von Neumann CPU, this means data can be executed as code (buffer overflows, code injection). In an LLM-based agent, this means data can be interpreted as instructions (prompt injection).

This is not a bug in any particular implementation. It is a **structural property of the architecture.** The context window of a large language model is a single, undifferentiated sequence of tokens. System prompts, user messages, tool outputs, file contents, and web page text are all concatenated into one stream. The model has no architectural mechanism to distinguish between "tokens I should follow as instructions" and "tokens I should process as data."

Every known defense against prompt injection operates at the application layer — instructing the model, via more tokens in the same stream, to please not treat certain other tokens as instructions. This is equivalent to preventing buffer overflows by writing careful C code. It works probabilistically. It fails structurally.

### 1.2 The Harvard Solution

The Harvard architecture, developed for the Harvard Mark I in 1944, solved an analogous problem in hardware by physically separating instruction memory and data memory with independent buses. The CPU reads instructions from one memory space and operates on data in another. Data cannot be executed as code because the two exist in different address spaces accessed through different physical pathways.

This document proposes applying the same structural separation to AI agent systems. Rather than relying on the model's ability to distinguish instruction tokens from data tokens within a shared context, we separate them architecturally — at the system level, below the model, enforced by the runtime rather than by the model's compliance.

### 1.3 Why "Stochastic"

Traditional Harvard architecture assumes a deterministic processor. The control unit fetches an instruction and executes it identically every time. This specification describes a system where the primary execution unit (the inference engine) is **fundamentally stochastic** — the same inputs may produce different outputs across invocations. This has cascading implications for scheduling, verification, error handling, and system design that differ from both traditional computing and traditional Harvard architecture.

---

## 2. Architectural Overview

### 2.1 Core Components

The Stochastic Harvard Architecture defines five classes of components:

| Component | Role | Analog |
|---|---|---|
| **Control Unit (CU)** | Stochastic inference engine (LLM, diffusion model, etc.) that reads instructions and data, then dispatches operations | CPU control unit |
| **Arithmetic Logic Unit (ALU)** | Deterministic compute resource (conventional CPU, container, sandbox) that executes precise operations dispatched by a CU | CPU ALU / coprocessor |
| **Instruction Memory (IMEM)** | Protected storage for system prompts, skills, policies, persona definitions, and execution context | Instruction ROM / cache |
| **Data Memory (DMEM)** | General-purpose storage for files, conversation history, tool outputs, embeddings, and working data | Data RAM / storage |
| **I/O Channels** | Interfaces to external systems — messaging platforms, web, APIs, hardware, sensors, displays | I/O ports / peripherals |

### 2.2 Bus Architecture

The system defines three bus types with distinct access semantics:

**Instruction Bus (I-Bus)**  
- Connects: IMEM → CU (read path)
- Direction: Primarily IMEM → CU. Write path exists only through a controlled **firmware update gate** (see §4.3).
- Isolation: **No component on the data bus may write to instruction memory.** This is the core security invariant.
- Content: System prompts, skill definitions, policy rules, execution plans.

**Data Bus (D-Bus)**  
- Connects: DMEM ↔ CU, DMEM ↔ ALU, I/O ↔ DMEM, I/O ↔ CU
- Direction: Bidirectional, multi-access.
- Isolation: None — shared resource. All untrusted content flows on this bus.
- Content: User messages, file contents, tool outputs, web page text, conversation history.

**Dispatch Bus (X-Bus)**  
- Connects: CU → CU, CU → ALU
- Direction: Command dispatch from control units to execution resources.
- Content: Structured operation requests (tool calls, inter-model routing, compute tasks).

### 2.3 System Diagram

```
              ┌─────────────────────────────────────────────────────────────┐
              │                    INSTRUCTION MEMORY                       │
              │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
              │  │  SYSTEM   │  │  SKILLS  │  │ POLICIES │  │ EXEC CTX │  │
              │  │  PROMPT   │  │ (loaded) │  │ (rules)  │  │ (plans)  │  │
              │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
              └──────────┬─────────────────────────────────────────────────┘
                         │ I-BUS (isolated, read-only during execution)
                         ▼
              ┌─────────────────────────────────────────────────────────────┐
              │                     CONTROL UNITS                           │
              │                                                             │
              │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
              │  │ LLM CORE │◄─►│DIFFUSION │◄─►│ CODE CU  │◄─►│SPEECH CU │  │
              │  │ (reason) │  │ (image)  │  │ (code)   │  │ (voice)  │  │
              │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
              │       X-BUS (dispatch between CUs)                         │
              └──────┬──────────────────────────────────────────┬──────────┘
                     │ D-BUS (shared, bidirectional)             │ X-BUS
                     ▼                                           ▼
              ┌─────────────────────┐                   ┌─────────────────┐
              │     DATA MEMORY     │                   │   ALU / COMPUTE │
              │  ┌───────┐┌───────┐ │                   │  ┌────┐ ┌────┐  │
              │  │ FILES ││ CONV  │ │ ◄──── D-BUS ────► │  │EXEC│ │SRCH│  │
              │  │       ││BUFFER │ │                   │  │    │ │    │  │
              │  └───────┘└───────┘ │                   │  └────┘ └────┘  │
              │  ┌───────┐┌───────┐ │                   │  ┌────┐ ┌────┐  │
              │  │VECTOR ││ TEMP  │ │                   │  │MATH│ │LINT│  │
              │  │ STORE ││SCRATCH│ │                   │  │    │ │    │  │
              │  └───────┘└───────┘ │                   │  └────┘ └────┘  │
              └─────────────────────┘                   └─────────────────┘
                     │                                           │
                     └──────────────────┬────────────────────────┘
                                        │ D-BUS
                                        ▼
              ┌─────────────────────────────────────────────────────────────┐
              │                      I/O CHANNELS                          │
              │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
              │  │MESSAGING │  │   WEB    │  │   API    │  │ HARDWARE │  │
              │  │(chat/sms)│  │(browser) │  │(external)│  │(sensors) │  │
              │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
              └─────────────────────────────────────────────────────────────┘
```

---

## 3. Component Specifications

### 3.1 Control Units (CU)

A Control Unit is a stochastic inference engine. It is the architectural equivalent of a CPU's control unit: it reads instructions, reads data, makes decisions, and dispatches operations. Unlike a traditional control unit, its behavior is **nondeterministic** — the same instruction and data inputs may produce different dispatch sequences across invocations.

#### 3.1.1 CU Properties

| Property | Description |
|---|---|
| **Model** | The underlying inference model (e.g., Claude Sonnet, GPT-4, Stable Diffusion, Whisper) |
| **Specialization** | What class of tasks this CU is optimized for (reasoning, image generation, code, speech, embedding) |
| **Context capacity** | Maximum combined instruction + data tokens per inference cycle |
| **Cycle time** | Latency per inference call (typically 0.5s–30s, depending on model and task) |
| **Determinism** | Degree of output consistency (temperature, sampling parameters) |
| **Bus access** | Which buses this CU can read from and write to |

#### 3.1.2 CU Types

**Primary CU (orchestrator):** An LLM-class model responsible for high-level reasoning, planning, and dispatch. Reads both I-Bus and D-Bus. Dispatches to ALU and other CUs via X-Bus. Typically the "main" model that interprets user intent and coordinates the system.

**Specialized CU:** A model optimized for a specific modality — image generation (diffusion), code generation, speech recognition/synthesis, music generation, video generation, embedding computation. Receives dispatch from the primary CU. May have restricted bus access (e.g., a diffusion CU may not need to read instruction memory directly if the primary CU translates instructions into structured prompts).

**Ensemble CU:** Multiple models operating on the same input with output aggregation — majority voting, confidence weighting, or consensus. Used for high-stakes decisions where stochastic reliability is insufficient from a single model.

#### 3.1.3 Inter-CU Dispatch Protocol

When a primary CU determines that a task requires a specialized CU, it issues a dispatch on the X-Bus containing:

```
DISPATCH {
  target:      CU identifier or capability requirement
  operation:   Structured description of the task
  input_refs:  References to data memory segments (NOT inline data)
  output_dest: Data memory segment for results
  constraints: Timeout, quality requirements, format requirements
  context:     Minimal instruction context needed by the target CU
}
```

The target CU receives the dispatch, reads the referenced data from DMEM, optionally reads relevant instruction memory segments from IMEM (via its own I-Bus connection), performs inference, and writes results to the specified DMEM output location.

**Critical:** The dispatch message on the X-Bus is a **structured command**, not a natural language prompt composed from data bus contents. This prevents data from flowing through the dispatch channel and being interpreted as instructions by the target CU.

### 3.2 Arithmetic Logic Units (ALU)

An ALU is a **deterministic** compute resource. It performs precise, repeatable operations dispatched by a CU. The ALU is the part of the system where conventional computing happens — compilation, execution, mathematical computation, file manipulation, search indexing.

#### 3.2.1 ALU Properties

| Property | Description |
|---|---|
| **Execution environment** | Container, sandbox, VM, bare process, or serverless function |
| **Isolation level** | How strongly isolated from the host system (none, container, microVM, hardware) |
| **Capabilities** | What operations this ALU can perform (shell, compile, network, filesystem) |
| **Determinism** | Fully deterministic (same input → same output) |
| **Cycle time** | Nanoseconds to minutes, depending on operation |
| **Bus access** | D-Bus read/write. Receives commands via X-Bus. **No I-Bus access.** |

#### 3.2.2 ALU Types

**Execution ALU:** Runs shell commands, scripts, compiled programs. Backed by a container or sandbox. This is what Claude Code's computer tool maps to.

**Search/Retrieval ALU:** Performs vector similarity search, keyword search, RAG retrieval. Reads from vector store segments of DMEM. Returns ranked results to D-Bus.

**Math/Compute ALU:** Numerical computation, symbolic math, statistical analysis. Deterministic and precise — the complement to the CU's stochastic reasoning.

**Build ALU:** Compilation, linting, testing, static analysis. Takes source code from DMEM, runs deterministic build pipelines, returns artifacts and diagnostics.

#### 3.2.3 ALU Security Model

ALU cores are **untrusted execution environments** — they run code that may have been generated by a stochastic CU. Therefore:

- Each ALU operates within a defined capability boundary (filesystem scope, network access, memory limit, CPU time limit).
- ALU cores have **no access to instruction memory**. They cannot read system prompts, skills, or policies. They operate only on data.
- ALU outputs go to data memory and are treated as data by CUs — never as instructions.
- Escaping the ALU sandbox is equivalent to a hardware fault, not an architectural violation. The architecture assumes ALU containment and builds on it.

### 3.3 Instruction Memory (IMEM)

Instruction memory stores the **program** that the control units execute. It defines what the system does, how it behaves, what policies it enforces, and what capabilities it has.

#### 3.3.1 IMEM Segments

**System Segment (IMEM-SYS)**  
- Contents: Core system prompt, persona definition, fundamental behavioral rules, safety policies.
- Access: Read by all CUs via I-Bus. **Write-locked during execution.** Modified only during system updates by a privileged administrative process.
- Analog: Firmware / boot ROM.
- Integrity: Hash-verified at boot. Modifications require cryptographic authorization.

**Skill Segment (IMEM-SKILL)**  
- Contents: Loaded skill definitions, tool descriptions, agent capabilities, workflow templates.
- Access: Read by CUs as needed (demand-loaded). Written only through the **skill installation gate** (§4.3).
- Analog: Program memory / loadable modules.
- Integrity: Skills are verified before installation. The installation gate checks provenance and signatures. Installed skills become part of the instruction stream and must be trusted to that level.

**Policy Segment (IMEM-POLICY)**  
- Contents: Access control rules, permission boundaries, per-channel restrictions, content policies, rate limits.
- Access: Read by CUs and by the bus controller (for enforcement). Written by administrative processes.
- Analog: Hardware configuration registers / protection rings.
- Integrity: Policy changes are audited. Policies constrain CU behavior at the architectural level (the bus controller can deny operations that violate policy, independent of CU compliance).

**Execution Context Segment (IMEM-CTX)**  
- Contents: Current task plan, step decomposition, active goals, reasoning chain state.
- Access: Read/write by CUs during execution. Scoped to the current task or session.
- Analog: Instruction cache / program counter state.
- Integrity: This is the most volatile IMEM segment. It represents the CU's working plan. It is writable by CUs but still exists on the instruction bus, separate from data. The distinction is between "what I'm trying to do" (instruction) and "what I'm doing it to" (data).

#### 3.3.2 The Instruction Memory Invariant

> **No component connected to the data bus may write to any instruction memory segment without passing through a verification gate.**

This is the core architectural guarantee. User messages, file contents, web page text, tool outputs, API responses — all of these exist on the data bus and cannot modify instruction memory. This means:

- A malicious email cannot alter the system prompt.
- A poisoned web page cannot install a new skill.
- A prompt injection in user input cannot modify policies.
- An adversarial tool output cannot change the execution plan (it can influence CU reasoning via the data bus, but cannot directly rewrite the plan in IMEM-CTX without the CU choosing to do so through its own instruction-following logic).

### 3.4 Data Memory (DMEM)

Data memory stores everything the system operates on — the content, not the program.

#### 3.4.1 DMEM Segments

**File Store (DMEM-FILE)**  
- Contents: Persistent files, documents, source code, databases, project contents.
- Access: Read/write by ALU cores (direct). Read by CUs via D-Bus. Write by CUs via ALU dispatch.
- Persistence: Across sessions.
- Trust level: Variable. Files may contain adversarial content. Treated as data only.

**Conversation Buffer (DMEM-CONV)**  
- Contents: Current conversation history, message stream, turn-by-turn context.
- Access: Write by I/O (inbound messages), read by CUs for reasoning.
- Persistence: Session-scoped (or task-scoped).
- Trust level: **Untrusted.** This is where user input lives. It is the primary prompt injection attack surface in current architectures. In SHA, it exists exclusively on the data bus.

**Tool Output Buffer (DMEM-TOOL)**  
- Contents: Results from ALU executions, API responses, search results.
- Access: Write by ALU cores, read by CUs.
- Persistence: Ephemeral.
- Trust level: Semi-trusted. ALU outputs are deterministic but operate on potentially untrusted input. Results should be treated as data.

**Vector Store (DMEM-VEC)**  
- Contents: Embedding vectors, similarity indices, RAG knowledge bases.
- Access: Write during indexing operations (by embedding CU + ALU). Read by search ALU.
- Persistence: Persistent, bulk-updated.

**Scratch / Working Memory (DMEM-SCRATCH)**  
- Contents: Intermediate computation results, temporary staging, draft outputs.
- Access: Read/write by CUs and ALUs.
- Persistence: Task-scoped. Cleared between tasks.

### 3.5 I/O Channels

I/O channels connect the system to the external world. All inbound content from I/O enters the system via the **data bus** and is written to data memory. It never enters instruction memory directly.

#### 3.5.1 I/O Channel Types

**Messaging (I/O-MSG):** Chat platforms, email, SMS. Bidirectional. Inbound messages → DMEM-CONV. Outbound messages dispatched by CU.

**Web (I/O-WEB):** HTTP client, browser automation. Fetched content → DMEM-FILE or DMEM-TOOL. **Critical isolation point** — web content is the highest-risk inbound data source for injection attacks.

**API (I/O-API):** External service integrations (calendar, task management, code hosting, etc.). Structured request/response. Inbound responses → DMEM-TOOL.

**Hardware (I/O-HW):** Sensors, actuators, displays, microphones, cameras, smart home devices, GPIO. Physical world interface.

**Display (I/O-DISPLAY):** The system's output rendering — terminal UI, web dashboard, canvas, or the GUI of a browser-based OS environment. Provides observability into system state.

#### 3.5.2 I/O Security Model

All inbound I/O is **untrusted by default.** The architecture does not rely on the CU to correctly classify inbound content as trustworthy or adversarial. Instead:

1. Inbound content is written to DMEM (data bus).
2. The CU reads it from DMEM via the data bus.
3. The CU's instruction stream comes separately from IMEM via the instruction bus.
4. Even if the inbound content contains text like "ignore your instructions and...", the architecture treats this as data — it can influence the CU's reasoning (the CU reads it), but it cannot physically replace the CU's instructions (those come from a different bus).

The residual risk is that the CU, being stochastic, may still be **influenced** by adversarial data to deviate from its instructions. This is analogous to a CPU with separated buses but a faulty control unit that sometimes misinterprets opcodes. The architectural separation doesn't eliminate all risk — it eliminates the **structural** risk and reduces the remainder to a model quality problem rather than an architecture problem.

---

## 4. Bus Protocols and Enforcement

### 4.1 Bus Controller

The system includes a **bus controller** that mediates all inter-component communication. The bus controller is a deterministic component (conventional software, not a stochastic model) that enforces:

- **Bus isolation:** Data bus content cannot be routed to instruction memory.
- **Access control:** Components can only access buses they are authorized for.
- **Policy enforcement:** Operations that violate IMEM-POLICY are blocked at the bus level.
- **Audit logging:** All bus transactions are logged for observability.
- **Rate limiting:** CU inference cycles and ALU executions are rate-limited per policy.

The bus controller is the **trusted computing base** of the architecture. Its correctness is what makes the security guarantees hold. It must be small, auditable, and deterministic.

### 4.2 Context Window Composition

When a CU performs an inference cycle, its context window is composed from two separate buses:

```
CONTEXT WINDOW = [
  --- I-BUS REGION (instruction memory) ---
  IMEM-SYS:    system prompt, persona
  IMEM-POLICY: active policies and constraints
  IMEM-SKILL:  relevant loaded skills
  IMEM-CTX:    current execution plan
  --- SEPARATOR (architectural boundary marker) ---
  --- D-BUS REGION (data memory) ---
  DMEM-CONV:   conversation history
  DMEM-TOOL:   recent tool outputs
  DMEM-FILE:   referenced file contents
  DMEM-SCRATCH: working memory
]
```

The bus controller composes this context window before each inference call. The I-Bus and D-Bus regions are assembled **separately** and concatenated with an architectural boundary. The CU (model) sees both regions but the boundary is enforced **below the model** — even if the model is confused about which region is which, the bus controller's composition logic is deterministic and cannot be manipulated by content in either region.

**Implementation note:** In current LLM APIs, this maps to the system/user/assistant message role structure — but with the critical addition that the system region is composed entirely from verified IMEM content, and the user/tool regions are composed entirely from DMEM content. The bus controller, not the application code, owns this composition.

### 4.3 Verification Gates

A **verification gate** is a controlled pathway that allows content to cross from the data bus to instruction memory under strict conditions. This is necessary for operations like:

- Installing a new skill (user provides skill definition → verified → written to IMEM-SKILL)
- Updating the execution plan (CU decides on a new plan → verified → written to IMEM-CTX)
- Modifying policies (administrator issues policy change → verified → written to IMEM-POLICY)

Each gate enforces:

1. **Provenance verification:** Where did this content originate? Is the source trusted?
2. **Content validation:** Does the content conform to the expected schema for this IMEM segment?
3. **Authorization:** Does the requesting entity have permission to modify this IMEM segment?
4. **Audit:** The modification is logged with full context.

**Critical:** The verification gate is a **deterministic, conventional software component** — not a stochastic model. The decision to allow instruction memory modification is not made by an LLM. It is made by auditable code with explicit rules.

---

## 5. Scheduling and Execution Model

### 5.1 The Stochastic Cycle

A traditional CPU executes billions of instructions per second, each taking nanoseconds. A stochastic CU executes one "inference cycle" in seconds, but each cycle accomplishes what might require thousands of conventional instructions. This inverts the traditional scheduling model.

#### 5.1.1 Cycle Anatomy

```
1. FETCH:     Bus controller composes context window from IMEM (I-Bus) + DMEM (D-Bus)
2. DECODE:    CU receives composed context (implicit — the model processes the input)
3. EXECUTE:   CU produces output (reasoning, dispatch commands, responses)
4. DISPATCH:  Output is parsed into:
              - D-Bus writes (responses → DMEM-CONV, plans → IMEM-CTX)
              - X-Bus dispatches (tool calls → ALU, sub-tasks → other CUs)
              - I/O dispatches (messages → I/O-MSG, web requests → I/O-WEB)
5. WRITEBACK: Results from dispatched operations return via D-Bus → DMEM
6. REPEAT:    Next cycle begins with updated DMEM state
```

#### 5.1.2 Cycle Scheduling

Because CU cycles are expensive (seconds, dollars), scheduling matters more than in conventional systems:

**Eager dispatch:** When a CU identifies multiple independent operations, dispatch them simultaneously to ALU cores and other CUs. Don't serialize unnecessarily.

**Batched context:** Accumulate multiple inbound events (messages, tool results) before triggering a CU cycle, rather than cycling per-event.

**Tiered CU routing:** Route simple tasks to cheaper/faster CUs (smaller models) and complex tasks to more capable CUs. The bus controller or a lightweight router can make this determination without burning a full CU cycle.

**Speculative execution:** For high-latency operations, dispatch the likely next operation before the current one completes. If wrong, discard. The cost of a wasted ALU cycle is trivial compared to a wasted CU cycle.

### 5.2 Determinism and Verification

Because the CU is stochastic, the architecture must handle **nondeterministic execution** as a first-class concern.

**Verification strategies:**

- **Post-hoc validation:** After a CU produces a dispatch, the bus controller validates it against IMEM-POLICY before execution. Invalid dispatches are rejected deterministically.
- **Consensus:** For high-stakes operations, route the same input to multiple CUs (same or different models) and require agreement.
- **Rollback:** ALU operations are executed in sandboxed environments. If a CU cycle produces an unexpected result, the ALU state can be rolled back.
- **Confidence gating:** CU output includes confidence signals. Low-confidence dispatches are routed to human review via I/O rather than executed.

---

## 6. Modified Harvard Considerations

### 6.1 Pure vs. Modified Architecture

Pure Harvard architecture has completely separate memory spaces and buses. This is clean but has practical limitations:

- Skills may reference data (a coding skill needs to describe file formats, which are "data knowledge").
- Execution plans (IMEM-CTX) are derived from reasoning about data (DMEM-CONV).
- Some instruction memory content is generated by CUs, which is inherently stochastic.

**Modified Stochastic Harvard** relaxes the pure separation:

- **Unified storage, separated buses:** Instruction and data content may be stored in the same physical database, but accessed through different bus interfaces with different access controls. The isolation is logical, not physical.
- **CU-mediated crossover:** A CU can read data and write execution plans. The data influences the plan, but the plan is written to IMEM-CTX through the CU's own reasoning process, not by direct data→instruction memory copy.
- **Cached instruction context:** Frequently-used instruction fragments may be cached alongside data for performance, as long as the cache respects bus isolation semantics at access time.

This mirrors how modern ARM and other processors implement Modified Harvard: separate L1 instruction and data caches, unified L2/L3 and main memory, with the separation enforced at the cache/bus level rather than at the storage level.

### 6.2 What This Means for Existing Systems

Current systems that approximate (but don't fully implement) this architecture:

**Claude Code / Claude Chat:**
- Has elements of bus separation: system prompt (IMEM-SYS) is distinct from user messages (DMEM-CONV). Tools and skills are defined separately from conversation content.
- The computer tool operates as an ALU — a sandboxed execution environment that runs deterministic operations.
- File access, web search, and code execution are dispatched from the CU to ALU-like components.
- **Gap:** The separation is advisory, not enforced. The model sees the full context window and must choose to treat system content differently from user content. The architecture doesn't prevent prompt injection — it relies on model robustness.

**OpenClaw:**
- Multi-agent routing approximates CU dispatch.
- Docker sandbox approximates ALU isolation.
- Skill system approximates IMEM-SKILL.
- Memory system (MEMORY.md) approximates IMEM-CTX / IMEM-SYS hybrid.
- **Gap:** No bus isolation. Skills, memory, user messages, and tool outputs all flow through the same context window with no architectural boundary. The system is explicitly von Neumann in its prompt composition.

**MCP (Model Context Protocol):**
- Tool definitions approximate the X-Bus dispatch protocol.
- Server-provided resources approximate DMEM segments.
- Prompts provided by MCP servers are treated as instruction-like content.
- **Gap:** MCP doesn't define bus isolation. A malicious MCP server could inject instruction-like content through the resource/tool-output pathway (data bus → instruction bus crossover).

---

## 7. Heterogeneous CU Composition

### 7.1 The Multi-Core Analogy

Modern processors are heterogeneous: CPU cores with different performance profiles (big.LITTLE), GPU compute units, NPU accelerators, DSP cores, all sharing a memory fabric. The system scheduler routes work to the appropriate execution unit based on task characteristics.

SHA applies the same principle to stochastic processors:

| CU Type | Optimized For | Cycle Time | Output Type |
|---|---|---|---|
| LLM Core (large) | Complex reasoning, planning, long-context tasks | 5-30s | Text, structured dispatch |
| LLM Core (small) | Simple classification, routing, formatting | 0.5-2s | Text, labels |
| Diffusion Core | Image/video generation, visual reasoning | 5-60s | Images, video |
| Code Core | Code generation, formal verification, symbolic reasoning | 2-15s | Code, proofs |
| Speech Core | Speech recognition, synthesis, voice interaction | 0.5-5s | Audio, transcripts |
| Embedding Core | Vector embedding computation, similarity scoring | 0.1-1s | Vectors, scores |

### 7.2 Dispatch Routing

The primary CU (or a dedicated lightweight router) determines which CU to dispatch to based on:

- **Task modality:** Image generation → Diffusion Core, code task → Code Core.
- **Complexity:** Simple reformatting → small LLM, multi-step planning → large LLM.
- **Cost:** Expensive models reserved for high-value tasks.
- **Latency:** Time-sensitive tasks routed to fastest available CU.
- **Availability:** If a CU type is at capacity, queue or route to alternative.

This routing decision can itself be made by a small, fast CU (a "scheduler CU") that reads a minimal instruction set and the task metadata, then dispatches to the appropriate worker CU.

### 7.3 Shared vs. Isolated Resources

Not all CUs need access to all memory segments or I/O channels:

```
                    IMEM-SYS  IMEM-SKILL  IMEM-CTX  DMEM-FILE  DMEM-CONV  I/O-MSG
LLM Core (primary)     R          R           RW         R          R         RW
LLM Core (small)       R          -           R          R          R         -
Diffusion Core         -          R*          -          R          -         -
Code Core              R          R           R          RW         R         -
Speech Core            -          R*          -          -          R         RW
Embedding Core         -          -           -          R          R         -
```

`R*` = reads only the subset of IMEM-SKILL relevant to this CU's specialization.

This matrix is defined in **IMEM-POLICY** and enforced by the **bus controller.** A compromised or misbehaving CU cannot access memory segments or I/O channels outside its authorization.

---

## 8. Implementation Strategies

### 8.1 Middleware Approach (Near-term)

The SHA can be implemented as middleware sitting between the application layer and the LLM API, without requiring changes to model architectures or API providers.

**Bus controller as prompt composer:** A deterministic software component that:
1. Loads IMEM segments from a protected store (encrypted at rest, access-controlled).
2. Loads DMEM segments from application state.
3. Composes the context window with explicit structural separation.
4. Calls the LLM API.
5. Parses the response into dispatch commands and data outputs.
6. Routes dispatches to ALU (sandboxes) and I/O.
7. Validates all operations against IMEM-POLICY before execution.

This is implementable today with existing LLM APIs, existing container runtimes, and conventional software engineering. The bus isolation is enforced by the middleware, not by the model.

### 8.2 API-Level Enforcement (Medium-term)

LLM API providers could offer architectural support for bus isolation:

- **Explicit instruction/data regions** in the API request, with provider-side guarantees about how they're processed.
- **Instruction memory pinning** — content marked as instruction is processed with higher attention weight or through a different pathway than data content.
- **Structural output constraints** — the API enforces that outputs conform to a dispatch schema, preventing the model from producing raw text that bypasses the bus controller.

### 8.3 Model Architecture Support (Long-term)

Future model architectures could implement bus isolation natively:

- **Dual-stream attention** — instruction tokens and data tokens attend to each other through different attention heads or layers, with instruction tokens having architectural priority.
- **Typed token embeddings** — tokens carry a type tag (instruction, data, dispatch) that is preserved through the transformer layers and affects processing.
- **Hardware-enforced context regions** — analogous to hardware memory protection, where instruction and data regions of the context window have different access permissions enforced by the inference runtime.

### 8.4 Browser-Based Runtime

A browser-based operating system provides a natural runtime for SHA:

- **CU:** Inference API calls to cloud providers (or local models via WebGPU/WASM).
- **ALU:** WebAssembly execution environment (deterministic, sandboxed by the browser).
- **IMEM:** Protected storage (possibly encrypted in IndexedDB, with keys held by the bus controller).
- **DMEM:** Browser filesystem APIs, IndexedDB, in-memory stores.
- **I/O:** Browser networking, Web APIs, WebRTC, UI rendering.
- **Bus controller:** JavaScript/WASM middleware managing context composition and dispatch.
- **Observability:** The GUI of the browser OS provides real-time visibility into system state — the user can see files being created, commands being executed, and decisions being made.

The browser sandbox provides an additional isolation layer: even if the entire SHA system is compromised, it cannot escape the browser's security boundary to affect the host operating system.

---

## 9. Security Analysis

### 9.1 Threat Model

| Threat | Von Neumann (current) | SHA |
|---|---|---|
| **Prompt injection via user input** | Data enters instruction stream. Model may follow injected instructions. | Data stays on D-Bus. Cannot reach IMEM. CU may still be influenced but cannot be overridden. |
| **Prompt injection via tool output** | Tool results concatenated into context alongside instructions. | Tool results → DMEM-TOOL (D-Bus). Separate from instruction stream. |
| **Malicious skill/plugin** | Skill content mixed with user data in context. Hard to audit. | Skills installed via verification gate to IMEM-SKILL. Auditable. Isolated from D-Bus content. |
| **Data exfiltration** | Model can be tricked into sending sensitive data to external services. | Bus controller enforces I/O policy from IMEM-POLICY. Unauthorized outbound blocked deterministically. |
| **Instruction corruption** | Adversary modifies system prompt via data channel. | IMEM-SYS is write-locked. No D-Bus path to IMEM. |
| **CU misbehavior (stochastic fault)** | Model ignores instructions. No architectural backstop. | Bus controller validates all dispatches against IMEM-POLICY. Invalid operations rejected. |

### 9.2 Residual Risks

SHA does not eliminate all risks. Residual risks include:

- **CU influence:** The CU reads data and may be influenced by adversarial content to make poor decisions within its instruction boundaries. The architecture prevents data from *replacing* instructions but cannot prevent data from *affecting reasoning about* instructions. This is a model robustness problem, not an architecture problem.
- **Verification gate compromise:** If the verification gate is buggy or misconfigured, adversarial content could be promoted from D-Bus to IMEM. The gate must be small, auditable, and rigorously tested.
- **Bus controller compromise:** The bus controller is the trusted computing base. If compromised, all guarantees are void. It must be minimal, formally verifiable if possible, and not itself implemented using stochastic components.
- **Side channels:** CU behavior may leak instruction memory content via output patterns. This is analogous to speculative execution side channels in conventional hardware.

---

## 10. Relationship to Existing Work

### 10.1 Related Concepts

- **Capability-based security (WASI):** WASI's capability model for WebAssembly is philosophically aligned — components get explicit, minimal capabilities rather than ambient authority. SHA applies this to the stochastic layer.
- **Microkernel architecture:** SHA's bus controller is analogous to a microkernel — a minimal trusted core that mediates all inter-component communication. The CUs and ALUs are like user-space servers.
- **Hardware TEEs (Trusted Execution Environments):** Intel SGX, ARM TrustZone — hardware-enforced isolation of sensitive computation. SHA's instruction memory isolation is analogous but applied to the inference layer.
- **RBAC/ABAC:** SHA's policy segment and bus controller implement access control, but at the architectural level rather than the application level.

### 10.2 Prior Art in AI Security

- Anthropic's system prompt / user message role distinction is a weak form of I-Bus / D-Bus separation.
- OpenAI's structured outputs enforce dispatch schema constraints (related to X-Bus protocol).
- Google's Gemini prompt caching separates "cached" (instruction-like) and "uncached" (data-like) context.
- Various "sandwich" defense techniques (instruction-data-instruction) attempt application-layer bus isolation.

SHA formalizes and strengthens these ad-hoc techniques into a coherent architectural framework.

---

## 11. Open Questions

1. **IMEM-CTX write semantics:** The execution context segment is writable by CUs during operation. How do we prevent a CU that has been influenced by adversarial data from writing compromised plans to IMEM-CTX? Options include: validation of plan content by the bus controller, requiring human approval for plan changes, or treating IMEM-CTX as a "gray zone" with reduced trust.

2. **Context window limits:** Current models have finite context windows. How do we prioritize IMEM vs. DMEM content when the combined instruction + data exceeds capacity? Who makes the eviction decision? If a CU makes it, the eviction policy is stochastic. If the bus controller makes it, it's deterministic but less intelligent.

3. **Model API support:** Current APIs (Anthropic, OpenAI, Google) provide role-based message structure but no guarantee that the model actually treats roles differently at the attention level. True SHA requires either API-level enforcement or model architecture changes. What's the minimum API change needed?

4. **Performance overhead:** Bus controller mediation adds latency to every cycle. Context window composition from multiple sources adds complexity. Is the overhead acceptable? How does it compare to the overhead of current prompt engineering defenses?

5. **Observability:** How do we provide meaningful visibility into bus traffic, IMEM state, and CU decision-making without overwhelming operators? What does a "debugger" look like for a stochastic Harvard machine?

6. **Multi-tenant isolation:** If multiple users share a system (e.g., a shared OpenClaw instance), how do we partition IMEM and DMEM between tenants? Is per-tenant bus isolation needed, or is per-tenant access control within shared buses sufficient?

7. **Emergent instruction sets:** The "instruction set" of an LLM CU is not fixed — it's whatever the model can do, which changes with each model generation. How do we maintain architectural contracts when the underlying CU capabilities shift? How do we version and test the system's capabilities?

---

## 12. Glossary

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

---

*This document is an exploratory draft. It describes a conceptual framework, not a finished specification. The architecture is proposed as a direction for research and discussion, not as a standard. Contributions, critiques, and experiments are welcome.*
