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

## Status

The project is being set up. The design is described in `docs/specs/`, and architecture decisions are recorded in `docs/adr/`.

## License

[MIT](LICENSE)
