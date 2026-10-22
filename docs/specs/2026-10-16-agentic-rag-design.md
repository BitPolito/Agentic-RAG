# Agentic-RAG: Design Specification

- **Status:** Draft, under review
- **Date:** 2026-10-16
- **Owners:** @BitPolito/ai-ml
- **Related:** ADR-0001 to ADR-0004 in `docs/adr/`

## 1. Context

BitPolito Academy (`BitPolito/bitcoin-academy`) is a local-first study companion. Its core value is the RAG pipeline in `services/ai`. That pipeline does hierarchical ingestion, hybrid retrieval, reranking, parent expansion and an evidence pack with mandatory citations, and today it is coupled to the Bitcoin course domain. The 2026/27 strategy has four steps:

1. Consolidate the backend ("Agent Harness") and freeze UX work.
2. Use the local hardware for R&D on small language models only. No frontier models.
3. Extract a domain-agnostic engine: courses become namespaces, retrieval becomes a library, and prompts become configuration.
4. Build a second project on top of that engine.

This repository hosts both the extracted engine and the second project: an **AST-aware agentic RAG for coding tasks**, delivered as an open benchmark and technical report.

### 1.1 Findings from the Academy codebase (2026-10-15)

- **Embeddings, the dense index and the LLM all run inside a Node service**, `workers/qvac-service`.
  - It uses the QVAC SDK 0.9 with GTE-Large FP16, HyperDB and Qwen3-4B Q4_K_M on llama.cpp. All LLM calls are serialized behind a FIFO lock.
  - The Python side reaches it over HTTP. There is no FAISS and no Ollama.
- **The Python side runs everything around the dense index:**
  - BM25, stored as a pickle and rebuilt on every ingest.
  - Weighted fusion, 0.6 dense + 0.4 lexical.
  - A flashrank reranker, MMR, and parent expansion from SQL.
  - The `EvidencePack` schema.
- **The generic core is about 2.3k–2.6k LOC of Python**, plus about 3k–4k LOC of portable unit tests.
- **Blocking issues for extraction:**
  - The code has no interfaces: no Protocols or ABCs.
  - Config is read from the environment at import time, and models and clients are module-level singletons.
  - `pipeline.run` mixes ingestion with the application's document state machine.
  - Domain vocabulary sits in generic code: Bitcoin synonyms, book-specific cleaning regexes, and `action`/`past_exam` fields in the evidence pack.
  - The index lifecycle is incomplete: there is no delete route in QVAC and no BM25 pruning.
- **Current evaluation is weak.**
  - A query passes when the top fused score is 0.35 or higher.
  - There is no recall@k, MRR or nDCG.
  - Page metadata is unreliable on markdown sources.

## 2. Goals and non-goals

### Goals

- **G1.** A domain-agnostic Python RAG engine (`agentic_rag`) with explicit ports and swappable adapters. Academy must be able to adopt it without loss of retrieval quality.
- **G2.** An AST-aware code retrieval extension (`agentic_rag_code`) for Python and TypeScript repositories.
- **G3.** A reproducible open benchmark that relates code-retrieval quality to agent task completion on local small models:
  - Recall@K, Precision@K, MRR and nDCG.
  - Task Completion Rate (TCR).
  - Latency and VRAM.
- **G4.** A public technical report comparing naive text RAG with AST-aware RAG on the lab workstation (RTX 5080, 16 GB VRAM).
- **G5.** A repository that a student team can contribute to in parallel. That means self-contained issues, clear contracts, and CI that enforces the architecture.

### Non-goals

- Frontier or commercial hosted models. Everything runs locally.
- UI, gamification, authentication, or course and lesson management. These stay in Academy.
- A conference submission. An arXiv preprint is an optional stretch goal.
- Languages other than Python and TypeScript for AST parsing in this phase.

## 3. Approach

**Shared foundation, then two parallel tracks.** A short foundation phase fixes the contracts: types, ports, config and the evaluation harness. After that, two tracks proceed in parallel against the same interfaces:

- the **Engine** track, which extracts from Academy;
- the **Research** track, which builds Code-RAG.

Interface changes require an ADR.

Rejected alternatives:

- **Engine-first, then research.** This serializes the team and leaves the research contributors idle.
- **Research-first prototype.** This produces a second, divergent copy of the engine.

## 4. Repository structure

The repository is a uv workspace with one lockfile and Python 3.11.

```
Agentic-RAG/
├── packages/
│   ├── engine/              # distribution: agentic-rag      import: agentic_rag
│   └── code-rag/            # distribution: agentic-rag-code import: agentic_rag_code
├── services/
│   └── qvac-worker/         # Node 22 QVAC service, ported from Academy
├── research/
│   ├── benchmark/           # runners and metrics (IR metrics, TCR, latency/VRAM)
│   ├── datasets/            # build scripts and dataset cards; data is fetched, not committed
│   ├── experiments/         # one folder per experiment: config.yaml + results/*.json
│   └── report/              # technical report sources
├── docs/
│   ├── architecture.md
│   ├── adr/                 # NNNN-title.md
│   └── specs/
├── .github/                 # workflows, issue/PR templates, CODEOWNERS, dependabot
├── AGENTS.md  CONTRIBUTING.md  README.md  LICENSE (MIT)
└── pyproject.toml  uv.lock
```

## 5. Engine architecture (`agentic_rag`)

The engine uses a hexagonal (ports and adapters) design.

```
agentic_rag/
├── types/        # Document, Block, Chunk, ParentChunk, CitationAnchor,
│                 # EvidenceChunk, EvidencePack, Namespace, IngestResult
├── ports/        # typing.Protocol definitions (no implementations)
├── config/       # EngineConfig (pydantic-settings), no import-time env reads
├── ingest/       # parsers/ (pdf, pptx, docx; registry by extension), clean/
│                 # (boilerplate detection + pluggable rules), chunking/
│                 # (parent/child, injectable TokenCounter), sections/, pipeline.py
├── retrieval/    # HybridRetriever (weighted + RRF fusion), rerank, MMR,
│                 # parent expansion, query transforms (HyDE, rewrite), compression
├── evidence/     # EvidencePackBuilder (dedup, ordering, token budget)
├── generation/   # prompt registry (versioned), context formatting ([ref_N]),
│                 # citation parsing, strict mode, no-evidence gate
├── agent/        # declarative task graph: system prompt, tools, retrieval policy
├── eval/         # golden-set format, Recall@K, Precision@K, MRR, nDCG, runner
└── adapters/     # optional extras, each importable only if its extra is installed
    ├── qvac/        # [qvac]  Embedder + VectorStore + LLM over the QVAC HTTP contract
    ├── local/       # [local] sentence-transformers embeddings, llama.cpp LLM (GPU)
    ├── vector/      # [faiss] or [lancedb], chosen by ADR in milestone E3
    ├── lexical/     # BM25 with JSON persistence (no pickle), injectable tokenizer/synonyms
    ├── sql/         # [sql]   ParentStore on SQLAlchemy
    ├── cache/       # [redis] semantic cache
    └── memory/      # in-memory fakes for every port (used by tests and eval smoke)
```

### 5.1 Ports

The signatures below are indicative. They are finalized in milestone M1.

| Port | Responsibility |
|---|---|
| `Embedder` | `embed(texts) -> vectors`, `dimension`, `max_tokens` |
| `VectorStore` | `upsert(namespace, chunks, vectors)`, `search(namespace, vector, k, filters)`, `delete(namespace, doc_id)`, `drop(namespace)` |
| `LexicalIndex` | `add`, `search(namespace, query, k)`, `delete(namespace, doc_id)` |
| `ParentStore` | `put_many(parents)`, `get_many(ids)`, `delete(doc_id)` |
| `Reranker` | `rerank(query, candidates, k)` |
| `LLM` | `generate(messages, **opts)`, `stream(...)`, `generate_json(messages, schema)` |
| `Cache` | `get(namespace, query)`, `put(namespace, query, value)` |
| `TokenCounter` | `count(text)`, aligned with the active embedder |

### 5.2 Rules

- `types` and `ports` import nothing from `adapters`.
- `agentic_rag` never imports `agentic_rag_code`.
- These rules are enforced by import-linter in CI.
- Every component receives its dependencies through its constructor. No module-level singletons.
- Domain specifics are passed in as configuration. Examples: synonym maps, cleaning rules, prompt sets, persona strings.
- `course_id` becomes `namespace` everywhere.
- The `EvidencePack` is a public, versioned contract pinned by a contract test.
  - `action` becomes an optional `task` field.
  - `doc_name` holds the document name.
  - The citation label is a separate field.
- **Strict mode** rejects or flags an answer that has no valid `[ref_N]` citation. It does not fall back to "all chunks cited".

### 5.3 Data flow

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

Progress and status are reported through callbacks or events. The application maps them to its own state machine.

### 5.4 Error handling

- Adapters raise typed errors from `agentic_rag.errors`:
  - `BackendUnavailable`
  - `LlmDisabled`
  - `StructuredGenerationError`
  - `IndexNotFound`
- Pipelines degrade explicitly:
  - If dense search is unavailable, lexical search runs alone and a warning is attached to the result.
  - There is no silent fallback.
- User-facing strings never live in the engine.

### 5.5 QVAC worker

The Node service is ported unchanged and then completed. The work covers:

- a `DELETE` route for documents and workspaces;
- a documented HTTP contract, an OpenAPI or JSON schema under `services/qvac-worker/contract/`;
- indexing of every chunk type.

The `qvac` adapter is the only Python code that knows this contract.

## 6. Code-RAG architecture (`agentic_rag_code`)

- **Parsing.**
  - Tree-sitter grammars for Python and TypeScript.
  - It extracts modules, classes, functions and methods, with their signatures, docstrings and imports.
  - It also extracts call sites and references.
- **Symbol graph.**
  - Nodes are symbols and files.
  - Edges are `defines`, `calls`, `imports`, `inherits` and `references`.
  - The graph is persisted behind a `CodeGraphStore` port. The first adapter is SQLite or networkx, decided by ADR in R1.
- **Chunking.** One chunk per symbol, with its enclosing scope as the parent. Long bodies are split at statement boundaries.
- **Retrieval.**
  - `CodeRetriever` combines semantic search (code embeddings via the engine's `Embedder`), lexical search on identifiers, and graph expansion (callers, callees and definitions to depth *d*).
  - It returns a standard `EvidencePack` whose anchors carry file, symbol and line range.
- **Agent.**
  - An orchestrator-workers pattern built on `agentic_rag.agent`.
  - A planner reads the issue, a retrieval worker queries the code index, and a code-generation worker uses a local quantized code model and produces a patch.
  - Patches are run in a sandbox, a container with the repository's tests.

## 7. Evaluation and benchmark

- **Engine eval** (`agentic_rag.eval`)
  - Golden sets list `query`, `relevant_ids` (chunk or section ids), and optionally `ground_truth` and `keywords`.
  - Metrics are Recall@K, Precision@K, MRR and nDCG@K, plus latency.
  - Baselines are committed as metrics-only JSON.
  - The first golden set is ported from Academy (Mastering Bitcoin 3e, CC BY-SA 4.0), with `relevant_ids` re-annotated.
- **Code benchmark** (`research/benchmark`)
  - **Tasks** come from real GitHub issues in permissively licensed Python and TypeScript repositories, using SWE-bench-style selection. A task qualifies only if it has a fix commit and a failing-then-passing test.
  - **Annotations** list the gold files, symbols and AST paths touched by the fix.
  - **Retrieval metrics** are Recall@K and Precision@K over symbols and files.
  - **Agent metric** is TCR: the share of tasks whose hidden tests pass after the patch.
  - **System metrics** are latency and peak VRAM per stage on the RTX 5080.
  - **Experiment matrix** covers {naive text RAG, AST-aware RAG, AST+graph} × {2–3 local code models} × K.
- **Reproducibility.**
  - Every experiment is a `config.yaml` that the runner executes.
  - Results carry a git SHA, model hashes and hardware info.
  - Datasets are rebuilt from scripts. Only the dataset cards and task manifests are committed.

## 8. Milestones

Milestones carry no dates. Each one closes when its exit criteria are met.

| ID | Milestone | Exit criteria |
|---|---|---|
| M0 | Foundation | Workspace scaffold; CI with lint, typecheck, test and quality-gate green on `main`; protected `main`; AGENTS.md, CONTRIBUTING, templates, labels and board in place; ADR-0001–0004 accepted |
| M1 | Core contracts | `types`, `ports`, `EngineConfig` and in-memory fakes; an end-to-end toy pipeline passes on fakes; EvidencePack contract test |
| M2 | Evaluation harness | IR metrics and runner; golden-set format; reproducible baseline report on the toy set |
| E1 | Ingestion extraction | Parsers, cleaning, chunking, sections and `IngestPipeline`; chunk-count parity with Academy on the open corpus within a stated tolerance; ported tests green; Academy technical golden set re-annotated with section ids |
| E2 | Retrieval extraction | Hybrid retrieval, rerank, MMR, parent expansion, EvidencePackBuilder, strict mode; known Academy defects fixed; metrics ≥ Academy baseline |
| E3 | Adapters | `qvac` adapter plus the completed worker; `local` GPU adapter; vector store chosen by ADR; JSON BM25; SQL ParentStore; Redis cache; eval passes on both the QVAC and local stacks |
| E4 | Agent runtime | Versioned prompt registry; declarative task graph; LLM concurrency control; Academy's 8 study actions expressible as configuration |
| E5 | Engine v0.x release | API docs, changelog, semantic versioning; installable from a git tag |
| E6 | Academy migration | Academy depends on `agentic_rag[qvac]`, duplicated code removed, Academy CI green, Academy eval unchanged or better (cross-repo) |
| R1 | Code ingestion | Python and TypeScript tree-sitter parsing, symbol graph, AST chunker; a mid-size repository indexed and queryable |
| R2 | Code retrieval | CodeRetriever (semantic + lexical + graph); naive vs AST-aware comparison on a small hand-annotated set |
| R3 | Dataset | Reproducible task-building pipeline; dataset card and license checks; v0.1 with a stated task count |
| R4 | Agent and TCR | Orchestrator-workers agent; sandboxed test execution; TCR runner end to end on the dataset |
| R5 | Benchmark and report | Full experiment matrix run; latency and VRAM profile; public technical report reproducible from `research/experiments/` |

**Dependencies:**

- M0 → M1 → M2, then E1–E5 and R1–R5 in parallel.
- R2 needs E2.
- R4 needs E3 (`local`) and E4.
- E6 needs E5.

## 9. Engineering conventions

### Git and pull requests

- `main` is protected.
- Work happens on short-lived branches named `<type>/<issue>-<desc>`.
- Types: feat, fix, refactor, docs, test, chore, perf, research.
- To merge a branch:
  1. Open a pull request against `main` with `Closes #n`.
  2. Wait for a green `quality-gate` and one approval.
  3. Merge with a merge commit (`gh pr merge --merge --delete-branch`).
  4. The branch is deleted.
- Soft limit of about 400 changed lines per PR. Open drafts early.

### Commits

- Conventional Commits: `type(scope): summary`.
- Scopes: engine, ingest, retrieval, evidence, generation, agent, adapters, eval, code-rag, dataset, bench, qvac, ci, docs.

### Quality

- ruff for lint and format.
- mypy `--strict` on `packages/`.
- pytest with a coverage ratchet starting at 80%.
- import-linter contracts.
- `node --test` for the worker.
- `eval-smoke` on in-memory fakes.

### Change control

- Any change that touches retrieval includes a before/after eval summary in the PR.
- Any port or contract change needs an ADR.

### Labels

- `type/*`: feature, bug, refactor, docs, test, chore, research.
- `area/*`: ingest, retrieval, adapters, eval, agent, code-rag, dataset, benchmark, qvac, infra.
- `track/*`: foundation, engine, research.
- `priority/*`: p0, p1, p2.
- `size/*`: S, M, L.
- `status/*`: blocked, needs-design.
- Also `epic`, `good first issue` and `agent-ready`.

### Board

GitHub Projects v2.

- **Fields:**
  - Status: Backlog → Ready → In progress → In review → Done.
  - Track, Area, Priority, Size, Milestone.
- **Views:** by status, by milestone, and "Ready and unassigned".

### Issues

Every issue follows this template:

- Context
- Goal
- Scope and out of scope
- Files and interfaces
- Acceptance criteria (checklist)
- How to verify (exact commands)
- Dependencies (`Blocked by #n`)
- Size

**Definition of Ready:** the acceptance criteria are testable, the dependencies are closed, and the size is L or smaller. Anything larger becomes an epic with sub-issues.

`agent-ready` marks issues precise enough to hand to a coding agent. A human reviews the resulting PR.

### Ownership

- `@BitPolito/ai-ml` owns the repository through CODEOWNERS.
- The lead maintainer (@lucaosti) approves ADRs and milestone closures.

### Cadence

- A weekly async update on the board covers done, next and blocked.
- A milestone review happens when the exit criteria are met.

### Language

All code, documentation, issues, milestones and commits are written in English.

## 10. Risks

| Risk | Mitigation |
|---|---|
| Ports designed wrong before both tracks use them | M1 includes a toy pipeline on fakes plus a Code-RAG spike against the ports; changes go through ADRs |
| Parity with Academy drifts while Academy keeps evolving | Pin the Academy commit used as the source; port fixes deliberately; E6 re-baselines |
| QVAC SDK instability (native runtime, model downloads) | The QVAC adapter is optional; the `local` stack is the default for research; CI uses fakes |
| Dataset licensing and contamination | Allow-list of licenses; record repository commit and issue date; prefer issues after the models' training cutoffs |
| 16 GB VRAM limits model choice | Quantized 7–8B code models; one model resident per stage; measure and report VRAM |
| Student availability is bursty | Small, self-contained issues; Definition of Ready; any contributor can pick up any `Ready` issue |
