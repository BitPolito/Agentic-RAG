# 0002. uv workspace monorepo for engine, code-rag and research

- Status: Proposed
- Date: 2026-10-16

## Context

The repository hosts three things that evolve together:

- a reusable library;
- an AST-aware extension of that library, which brings heavy dependencies (tree-sitter grammars, GPU stacks);
- research code (benchmark, datasets, experiments).

We want a single CI and a single lockfile. Code-RAG dependencies must not leak into the engine install.

## Decision

The repository is a uv workspace, using Python 3.11.

| Package | Distribution | Import name |
|---|---|---|
| `packages/engine` | `agentic-rag` | `agentic_rag` |
| `packages/code-rag` | `agentic-rag-code` | `agentic_rag_code` |

- `agentic-rag-code` depends on `agentic-rag`.
- `research/` holds runners and experiment configs that depend on both packages. It is not a published package.
- `services/qvac-worker` is a Node 22 project with its own `package.json`.
- Tooling: ruff (lint and format), mypy `--strict` on `packages/`, pytest, and import-linter.

## Consequences

- Atomic changes can span the engine and code-rag.
- There is one lockfile, so environments are reproducible.
- Each package keeps its own dependency set and extras.
- CI has to run per-package jobs and a Node job.
- Releasing the engine needs package-scoped tags, for example `engine-v0.1.0`.
