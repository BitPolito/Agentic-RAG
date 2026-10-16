# agentic-rag-code

`agentic-rag-code` is the AST-aware code retrieval extension, imported as `agentic_rag_code`. It provides:

- tree-sitter parsing for Python and TypeScript;
- a symbol and call graph;
- symbol-level chunking;
- a code retriever that returns standard evidence packs.

It depends only on the public ports and types of `agentic_rag`. The design is described in `docs/specs/2026-10-16-agentic-rag-design.md` (section 6).
