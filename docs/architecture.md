# Architecture overview

This page is a one-page map of the repository. The full design is in [`docs/specs/2026-10-16-agentic-rag-design.md`](specs/2026-10-16-agentic-rag-design.md), and the decisions are recorded in [`docs/adr/`](adr/README.md).

## Packages

| Path | Distribution | Import | Role |
|---|---|---|---|
| `packages/engine` | `agentic-rag` | `agentic_rag` | Domain-agnostic RAG engine: types, ports, pipelines, evidence pack, evaluation, adapters |
| `packages/code-rag` | `agentic-rag-code` | `agentic_rag_code` | AST-aware code retrieval for Python and TypeScript, built on the engine's ports |
| `services/qvac-worker` | none (Node 22) | none | QVAC SDK service used by the `qvac` adapter (ported in milestone E3) |
| `research/` | none | none | Benchmark runners, dataset builders, experiments and the technical report |

## Engine layers

Each layer may import only the layers below it. import-linter enforces this in CI.

```
agentic_rag.adapters                       backends behind optional extras (qvac, local, vector, lexical, sql, cache, memory)
agentic_rag.ingest | retrieval | generation | agent | eval
agentic_rag.evidence                       EvidencePack construction
agentic_rag.ports                          typing.Protocol interfaces
agentic_rag.types                          domain-neutral data model
```

`agentic_rag` never imports `agentic_rag_code`. The dependency only goes the other way.

## Data flow

**Ingestion**

```
file -> Parser -> Document(blocks) -> Cleaner -> Chunker -> (parents, children)
     -> ParentStore.put_many(parents)
     -> Embedder.embed(children) -> VectorStore.upsert
     -> LexicalIndex.add(children)
     -> IngestResult(stats, section_tree, warnings)
```

**Query**

```
query -> QueryTransform* -> {VectorStore.search, LexicalIndex.search} -> Fusion
      -> Reranker -> MMR -> ParentStore expansion -> EvidencePackBuilder
      -> (optional) LLM with strict citation policy -> Answer(evidence, citations)
```

## Ports

The ports are:

- `Embedder`
- `VectorStore`
- `LexicalIndex`
- `ParentStore`
- `Reranker`
- `LLM`
- `Cache`
- `TokenCounter`

Their exact signatures are fixed by ADR-0005 (milestone M1). Every port has an in-memory fake in `agentic_rag.adapters.memory`, and a conformance suite that every adapter must pass.

## Tracks

- **Foundation (M0–M2):** workspace, contracts and the evaluation harness.
- **Engine (E1–E6):** extract ingestion, retrieval, adapters and the agent runtime from BitPolito Academy, release the engine, then migrate Academy onto it.
- **Research (R1–R5):** code ingestion, code retrieval, the benchmark dataset, the agent and its Task Completion Rate harness, and the technical report.
