# Architecture Decision Records

Each ADR records one significant decision: its context, the decision itself, and the consequences.

- **File name:** `NNNN-kebab-title.md`.
- **Status:** one of Proposed, Accepted, Superseded by NNNN, or Deprecated.
- **When an ADR is required:** any change to a port, a public contract (for example `EvidencePack` or the QVAC HTTP contract), the repository layout, or the core tooling.

| ADR | Title | Status |
|---|---|---|
| [0001](0001-ports-and-adapters.md) | Hexagonal architecture with ports and adapters | Proposed |
| [0002](0002-uv-workspace-monorepo.md) | uv workspace monorepo for engine, code-rag and research | Proposed |
| [0003](0003-qvac-as-optional-adapter.md) | QVAC as an optional adapter, not the engine runtime | Proposed |
| [0004](0004-copy-then-migrate-from-academy.md) | Copy from Academy now, migrate Academy later | Proposed |
