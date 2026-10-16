# 0003. QVAC as an optional adapter, not the engine runtime

- Status: Proposed
- Date: 2026-10-16

## Context

In Academy, the QVAC SDK (Node 22, llama.cpp, HyperDB) provides embeddings (GTE-Large), dense indexing and LLM inference (Qwen3-4B) behind an HTTP service. Its limits for this project:

- Embedding models cannot be chosen from Python.
- The service does not use the GPU.
- It lacks index deletion.
- It has had runtime stability issues: native runtime crashes, and long model downloads in CI.

The AST research needs code-specific embeddings and GPU-local models on an RTX 5080. Academy has to keep working on QVAC.

## Decision

- QVAC is one adapter set, `agentic_rag.adapters.qvac` behind the `[qvac]` extra. It implements `Embedder`, `VectorStore` and `LLM` over the documented HTTP contract.
- The Node worker is ported to `services/qvac-worker`. It gets a `DELETE` route and a versioned contract under `services/qvac-worker/contract/`.
- A `local` adapter set is the default for research: sentence-transformers embeddings, a llama.cpp LLM on the GPU, and a vector store chosen by ADR in milestone E3.
- CI does not depend on QVAC. Tests use in-memory fakes, plus a separate worker test job.

## Consequences

- Academy can adopt the engine with no behaviour change, through the QVAC adapter.
- Research is not limited by what the QVAC SDK supports.
- Two backend stacks have to be kept at parity. The eval suite has to pass on both (exit criterion of E3).
