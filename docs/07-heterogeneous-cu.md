# 7. Heterogeneous CU Composition

[< Modified Harvard Considerations](06-modified-harvard.md) | [Implementation Strategies >](08-implementation.md)

## 7.1 The Multi-Core Analogy

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

## 7.2 Dispatch Routing

The primary CU (or a dedicated lightweight router) determines which CU to dispatch to based on:

- **Task modality:** Image generation → Diffusion Core, code task → Code Core.
- **Complexity:** Simple reformatting → small LLM, multi-step planning → large LLM.
- **Cost:** Expensive models reserved for high-value tasks.
- **Latency:** Time-sensitive tasks routed to fastest available CU.
- **Availability:** If a CU type is at capacity, queue or route to alternative.

This routing decision can itself be made by a small, fast CU (a "scheduler CU") that reads a minimal instruction set and the task metadata, then dispatches to the appropriate worker CU.

## 7.3 Shared vs. Isolated Resources

Not all CUs need access to all memory segments or I/O channels:

| CU Type | IMEM-SYS | IMEM-SKILL | IMEM-CTX | DMEM-FILE | DMEM-CONV | I/O-MSG |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **LLM Core (primary)** | R | R | RW | R | R | RW |
| **LLM Core (small)** | R | - | R | R | R | - |
| **Diffusion Core** | - | R* | - | R | - | - |
| **Code Core** | R | R | R | RW | R | - |
| **Speech Core** | - | R* | - | - | R | RW |
| **Embedding Core** | - | - | - | R | R | - |

`R*` = reads only the subset of IMEM-SKILL relevant to this CU's specialization.

This matrix is defined in **IMEM-POLICY** and enforced by the **bus controller.** A compromised or misbehaving CU cannot access memory segments or I/O channels outside its authorization.
