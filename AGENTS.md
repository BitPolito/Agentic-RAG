# Instructions for automated contributors

These rules apply to any coding agent that works in this repository. Humans should read `CONTRIBUTING.md`, which these rules condense.

## Before you write code

1. Read the issue completely: Goal, Scope, Out of scope, Files, Acceptance criteria, How to verify.
2. Read `docs/specs/2026-10-16-agentic-rag-design.md`, at least sections 4, 5 and 9, and every ADR the issue references in `docs/adr/`.
3. If the issue is ambiguous or contradicts the spec, comment on the issue and stop. Do not guess.

## While you work

- Change only what the issue's Scope covers. Do not refactor, rename or "improve" anything outside it.
- Respect the architecture (ADR-0001):
  - core code depends only on `agentic_rag.types` and `agentic_rag.ports`;
  - backends live in `agentic_rag.adapters`;
  - dependencies are injected through constructors;
  - no module-level singletons;
  - no `os.getenv` or `os.environ` outside `EngineConfig`.
- Never change a port or a public contract without an accepted ADR.
- Write the tests first. Every acceptance criterion needs a test that proves it.
- All code, comments, docstrings and messages are written in English. Docstrings use the Google style.
- Do not add references to AI tools or assistants, or attribution trailers, in code, docs or commit messages.

## Before you open a pull request

Run every check, and the issue's own "How to verify" commands:

```bash
uv run ruff check . && uv run ruff format --check .
uv run mypy
uv run lint-imports
uv run pytest
```

Then:

- **Branch:** `<type>/<issue>-<short-description>`.
- **Commits:** Conventional Commits, `type(scope): summary`, with a `Refs #<issue>` footer.
- **Pull request:** against `main`, filling in the template, with `Closes #<issue>`. For retrieval changes, paste the before and after `arag-eval` SUMMARY.
- Never merge your own pull request. A human reviews and merges it.
