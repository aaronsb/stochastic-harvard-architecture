# 2. Architectural Overview

[< Motivation](01-motivation.md) | [Component Specifications >](03-components.md)

## 2.1 Core Components

The Stochastic Harvard Architecture defines five classes of components:

| Component | Role | Analog |
|---|---|---|
| **Control Unit (CU)** | Stochastic inference engine (LLM, diffusion model, etc.) that reads instructions and data, then dispatches operations | CPU control unit |
| **Arithmetic Logic Unit (ALU)** | Deterministic compute resource (conventional CPU, container, sandbox) that executes precise operations dispatched by a CU | CPU ALU / coprocessor |
| **Instruction Memory (IMEM)** | Protected storage for system prompts, skills, policies, persona definitions, and execution context | Instruction ROM / cache |
| **Data Memory (DMEM)** | General-purpose storage for files, conversation history, tool outputs, embeddings, and working data | Data RAM / storage |
| **I/O Channels** | Interfaces to external systems — messaging platforms, web, APIs, hardware, sensors, displays | I/O ports / peripherals |

## 2.2 Bus Architecture

The system defines three bus types with distinct access semantics:

**Instruction Bus (I-Bus)**
- Connects: IMEM → CU (read path)
- Direction: Primarily IMEM → CU. Write path exists only through a controlled **firmware update gate** (see [§4.3](04-bus-protocols.md#43-verification-gates)).
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

## 2.3 System Diagram

```mermaid
flowchart TD
    subgraph IMEM["INSTRUCTION MEMORY"]
        direction LR
        SYS["System\nPrompt"]
        SKILL["Skills\n(loaded)"]
        POL["Policies\n(rules)"]
        ECTX["Exec Ctx\n(plans)"]
    end

    subgraph CU["CONTROL UNITS"]
        direction LR
        LLM["LLM Core\n(reason)"]
        DIFF["Diffusion\n(image)"]
        CODE["Code CU\n(code)"]
        SPEECH["Speech CU\n(voice)"]
        LLM <-->|"X-Bus"| DIFF
        DIFF <-->|"X-Bus"| CODE
        CODE <-->|"X-Bus"| SPEECH
    end

    subgraph DMEM["DATA MEMORY"]
        direction LR
        FILES["Files"]
        CONV["Conv\nBuffer"]
        VEC["Vector\nStore"]
        SCRATCH["Temp\nScratch"]
    end

    subgraph ALU["ALU / COMPUTE"]
        direction LR
        EXEC["Exec"]
        SRCH["Search"]
        MATH["Math"]
        LINT["Lint"]
    end

    subgraph IO["I/O CHANNELS"]
        direction LR
        MSG["Messaging\n(chat/sms)"]
        WEB["Web\n(browser)"]
        API["API\n(external)"]
        HW["Hardware\n(sensors)"]
    end

    IMEM -->|"I-Bus (isolated, read-only)"| CU
    CU -->|"D-Bus (shared, bidirectional)"| DMEM
    CU -->|"X-Bus (dispatch)"| ALU
    DMEM <-->|"D-Bus"| ALU
    DMEM -->|"D-Bus"| IO
    ALU -->|"D-Bus"| IO

    style IMEM fill:#2D7D9A,stroke:#4A5568,color:#FFFFFF
    style SYS fill:#2D7D9A,stroke:#FFFFFF,color:#FFFFFF
    style SKILL fill:#2D7D9A,stroke:#FFFFFF,color:#FFFFFF
    style POL fill:#2D7D9A,stroke:#FFFFFF,color:#FFFFFF
    style ECTX fill:#2D7D9A,stroke:#FFFFFF,color:#FFFFFF

    style CU fill:#7B2D8E,stroke:#4A5568,color:#FFFFFF
    style LLM fill:#7B2D8E,stroke:#FFFFFF,color:#FFFFFF
    style DIFF fill:#7B2D8E,stroke:#FFFFFF,color:#FFFFFF
    style CODE fill:#7B2D8E,stroke:#FFFFFF,color:#FFFFFF
    style SPEECH fill:#7B2D8E,stroke:#FFFFFF,color:#FFFFFF

    style DMEM fill:#2D8E5E,stroke:#4A5568,color:#FFFFFF
    style FILES fill:#2D8E5E,stroke:#FFFFFF,color:#FFFFFF
    style CONV fill:#2D8E5E,stroke:#FFFFFF,color:#FFFFFF
    style VEC fill:#2D8E5E,stroke:#FFFFFF,color:#FFFFFF
    style SCRATCH fill:#2D8E5E,stroke:#FFFFFF,color:#FFFFFF

    style ALU fill:#5A6ABF,stroke:#4A5568,color:#FFFFFF
    style EXEC fill:#5A6ABF,stroke:#FFFFFF,color:#FFFFFF
    style SRCH fill:#5A6ABF,stroke:#FFFFFF,color:#FFFFFF
    style MATH fill:#5A6ABF,stroke:#FFFFFF,color:#FFFFFF
    style LINT fill:#5A6ABF,stroke:#FFFFFF,color:#FFFFFF

    style IO fill:#C2572A,stroke:#4A5568,color:#FFFFFF
    style MSG fill:#C2572A,stroke:#FFFFFF,color:#FFFFFF
    style WEB fill:#C2572A,stroke:#FFFFFF,color:#FFFFFF
    style API fill:#C2572A,stroke:#FFFFFF,color:#FFFFFF
    style HW fill:#C2572A,stroke:#FFFFFF,color:#FFFFFF
```
