# Agentic-RAG

Agentic-RAG is a local-first Retrieval-Augmented Generation engine built by the BitPolito AI/ML team. The repository has two parts:

- **`agentic_rag`**, a domain-agnostic RAG engine. It is extracted from [BitPolito Academy](https://github.com/BitPolito/bitcoin-academy) and covers:
  - hierarchical ingestion;
  - hybrid retrieval (dense and lexical);
  - reranking and parent expansion;
  - a citation-bearing evidence pack;
  - swappable backends through ports and adapters.
- **`agentic_rag_code`**, an AST-aware retrieval extension for Python and TypeScript codebases. It is evaluated through an open benchmark that relates retrieval quality to agent task completion on local small language models.

Everything runs locally. The project does not use frontier or hosted commercial models.

## Quick start

```bash
git clone https://github.com/BitPolito/Agentic-RAG.git
cd Agentic-RAG
uv sync
uv run pytest
```

## Documentation

- [Contributing guide](CONTRIBUTING.md): workflow, conventions and how to pick up work.
- [Architecture overview](docs/architecture.md): packages, layers and data flow.
- [Design specification](docs/specs/2026-10-16-agentic-rag-design.md): goals, architecture, milestones and conventions.
- [Architecture Decision Records](docs/adr/README.md).
- [Project board](https://github.com/orgs/BitPolito/projects/23).
- Instructions for automated contributors: [AGENTS.md](AGENTS.md).

## License

[MIT](LICENSE)
