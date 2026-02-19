# 8. Implementation Strategies

[< Heterogeneous CU Composition](07-heterogeneous-cu.md) | [Security Analysis >](09-security.md)

## 8.1 Middleware Approach (Near-term)

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

## 8.2 API-Level Enforcement (Medium-term)

LLM API providers could offer architectural support for bus isolation:

- **Explicit instruction/data regions** in the API request, with provider-side guarantees about how they're processed.
- **Instruction memory pinning** — content marked as instruction is processed with higher attention weight or through a different pathway than data content.
- **Structural output constraints** — the API enforces that outputs conform to a dispatch schema, preventing the model from producing raw text that bypasses the bus controller.

## 8.3 Model Architecture Support (Long-term)

Future model architectures could implement bus isolation natively:

- **Dual-stream attention** — instruction tokens and data tokens attend to each other through different attention heads or layers, with instruction tokens having architectural priority.
- **Typed token embeddings** — tokens carry a type tag (instruction, data, dispatch) that is preserved through the transformer layers and affects processing.
- **Hardware-enforced context regions** — analogous to hardware memory protection, where instruction and data regions of the context window have different access permissions enforced by the inference runtime.

## 8.4 Browser-Based Runtime

A browser-based operating system provides a natural runtime for SHA:

- **CU:** Inference API calls to cloud providers (or local models via WebGPU/WASM).
- **ALU:** WebAssembly execution environment (deterministic, sandboxed by the browser).
- **IMEM:** Protected storage (possibly encrypted in IndexedDB, with keys held by the bus controller).
- **DMEM:** Browser filesystem APIs, IndexedDB, in-memory stores.
- **I/O:** Browser networking, Web APIs, WebRTC, UI rendering.
- **Bus controller:** JavaScript/WASM middleware managing context composition and dispatch.
- **Observability:** The GUI of the browser OS provides real-time visibility into system state — the user can see files being created, commands being executed, and decisions being made.

The browser sandbox provides an additional isolation layer: even if the entire SHA system is compromised, it cannot escape the browser's security boundary to affect the host operating system.
