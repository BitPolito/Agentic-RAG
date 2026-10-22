# 0001. Hexagonal architecture with ports and adapters

- Status: Proposed
- Date: 2026-10-16

## Context

The Academy RAG code has no interfaces. Embeddings, vector search, the LLM, parent storage and lexical search are all hard-wired:

- to the QVAC HTTP API;
- to SQLAlchemy models;
- to files under `QVAC_INGEST_DIR`.

Configuration is read at import time. Models and clients are module-level singletons.

The engine has to serve two consumers with different backends:

- Academy, which uses QVAC;
- the Code-RAG research track, which needs GPU-local models and code embeddings.

## Decision

The engine is organised as ports and adapters.

- `agentic_rag.ports` defines `typing.Protocol` interfaces:
  - `Embedder`, `VectorStore`, `LexicalIndex`, `ParentStore`
  - `Reranker`, `LLM`, `Cache`, `TokenCounter`
- Core packages (`types`, `ingest`, `retrieval`, `evidence`, `generation`, `agent`, `eval`) depend only on ports.
- Every adapter lives under `agentic_rag.adapters` and sits behind an optional extra.
- Dependencies are injected through constructors.
- Configuration is a single `EngineConfig` object built with pydantic-settings.
- Every port has an in-memory fake in `adapters/memory`.
- import-linter enforces the dependency rules in CI.

## Consequences

- Backends can be swapped without touching pipeline code.
- Unit tests and the eval smoke test run without models or network access.
- More upfront design. Port changes need an ADR, which slows changes to the interfaces.
- Academy code has to be refactored while it is being ported. Lifting it as-is is not possible.
