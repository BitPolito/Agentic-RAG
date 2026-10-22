# Contributing to Agentic-RAG

This guide covers everything you need to go from cloning the repository to getting a pull request merged. The design behind these rules is in `docs/specs/2026-10-16-agentic-rag-design.md`, and the architecture decisions are in `docs/adr/`.

All code, comments, documentation, issues, commit messages and pull requests are written in **English**.

## 1. Setup

You need [uv](https://docs.astral.sh/uv/). uv installs Python 3.11 for you, following `.python-version`.

```bash
git clone https://github.com/BitPolito/Agentic-RAG.git
cd Agentic-RAG
uv sync
uv run pytest
```

Always run project commands through `uv run`, so they use the project's virtual environment.

## 2. Quality checks

These are the same checks CI runs. Run them before you push:

```bash
uv run ruff check .
uv run ruff format --check .
uv run mypy
uv run lint-imports
uv run pytest
```

| Check | What it enforces |
|---|---|
| ruff | Lint and formatting, Python 3.11, line length 100, Google-style docstrings. Docstrings are not required in tests. |
| mypy | Strict typing on `packages/`. |
| import-linter | The architecture rules from ADR-0001: types and ports never import adapters; the engine never imports `agentic_rag_code`; engine modules respect the layer order. |
| pytest | Tests with coverage. The build fails below 80%. |

CI runs every check on each pull request. The `quality-gate` job aggregates them, and it is the only required status check.

## 3. Picking up work

1. **Find an issue.** Open the [project board](https://github.com/orgs/BitPolito/projects/23) and choose an issue in the **Ready** column.
   - Issues labelled `good first issue` are a good starting point.
   - Do not start an issue in **Backlog**. Its blockers are not closed yet.
2. **Claim it.** Assign the issue to yourself and move it to **In progress**.
3. **Read before coding.** Read the issue's Goal, Scope, Out of scope and Acceptance criteria. Read the spec sections and ADRs it references.
4. **Ask early.** If anything is unclear, ask in the issue. Do not guess.

### Definition of Ready

An issue is **Ready** only when all of these hold:

- its acceptance criteria are testable;
- every issue it is blocked by is closed;
- its size is `S`, `M` or `L`.

Work larger than `L` is turned into an epic with sub-issues.

## 4. Branches

- `main` is protected. Never commit to it directly.
- Create one short-lived branch per issue, named `<type>/<issue-number>-<short-description>`. For example: `feat/12-hybrid-retriever`, `fix/31-bm25-delete`.
- Branch types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `research`.
- A branch is deleted after its pull request is merged.

## 5. Commits

Use [Conventional Commits](https://www.conventionalcommits.org/), in the form `type(scope): summary`.

- **Summary:** imperative mood, at most about 72 characters.
- **Body:** explains *why*, when the reason is not obvious.
- **Footer:** `Refs #<issue>`.

Allowed scopes: `engine`, `ingest`, `retrieval`, `evidence`, `generation`, `agent`, `adapters`, `eval`, `code-rag`, `dataset`, `bench`, `qvac`, `ci`, `docs`.

Example:

```
feat(retrieval): add reciprocal rank fusion

Weighted fusion is sensitive to score scales across backends; RRF is
scale-free and is needed for the QVAC and local stacks to be comparable.

Refs #18
```

## 6. Pull requests

1. **Open a draft early.** Open a pull request against `main` as soon as you have something to show. Keep it a draft until it is ready.
2. **Fill in the PR template.** It covers What, Why, `Closes #<issue>`, how you verified the change, and reviewer notes.
3. **Keep it small.** Aim for about 400 changed lines or fewer. Split anything larger.
4. **Include eval results for retrieval changes.** If the PR changes retrieval behaviour (ingestion, chunking, search, fusion, reranking or the evidence pack), paste the `arag-eval` SUMMARY block from before and after the change.
5. **Request review.** A review is requested automatically from `@BitPolito/ai-ml` (see `CODEOWNERS`). Move the issue to **In review**.
6. **Address feedback with new commits.** Do not force-push a branch that is under review.
7. **Merge.** A pull request is merged when `quality-gate` is green and it has at least one approval. Always merge with a merge commit, never squash or rebase, and delete the branch:

   ```bash
   gh pr merge <number> --merge --delete-branch
   ```

8. **Close the issue.** `Closes #<issue>` in the PR closes the issue automatically. If the PR only covers part of the issue, use `Refs #<issue>` and comment on the issue with the progress.

## 7. Architecture Decision Records

An ADR in `docs/adr/` is required, before the implementation, for any change to:

- a port (`agentic_rag.ports`) or a public contract, such as `EvidencePack` or the QVAC HTTP contract;
- the repository layout or the core tooling;
- a technology choice that other code will depend on, such as the vector store or the code graph store.

Copy the format of an existing ADR. Open it with status **Proposed**. It becomes **Accepted** once its pull request is merged. The lead maintainer approves ADRs.

## 8. Porting code from BitPolito Academy

The engine is extracted from `BitPolito/bitcoin-academy` (ADR-0004). When you port code:

- Copy from the pinned commit named in the backlog (`b78d661`). Record the source in the PR as `Source: bitcoin-academy@<sha> <path>`.
- Port the matching tests together with the code.
- Remove domain-specific code while porting: course and Bitcoin vocabulary, book-specific rules, user-facing strings. Pass these in as configuration instead.
- Do not modify Academy. Its migration is milestone E6.

## 9. Labels

| Label | Meaning |
|---|---|
| `type/*` | Kind of change: feature, bug, refactor, docs, test, chore, research. |
| `area/*` | Part of the system: ingest, retrieval, adapters, eval, agent, code-rag, dataset, benchmark, qvac, infra. |
| `track/*` | Foundation (M0–M2), engine (E1–E6) or research (R1–R5). |
| `priority/p0`–`p2` | p0 blocks its milestone, p1 is needed for it, p2 is nice to have. |
| `size/S`, `size/M`, `size/L` | S is up to about half a day, M is one to three days, L is about a week. |
| `status/blocked`, `status/needs-design` | Waiting on something, or needs an ADR first. |
| `epic` | Tracks the sub-issues of a later milestone. |
| `agent-ready` | Precise enough to hand to a coding agent. A human still reviews the resulting PR. |
| `good first issue` | A good starting point for newcomers. |

## 10. Cadence

- **Weekly update:** each contributor posts a short async update on their issue or in the team channel, covering done, next and blocked.
- **Milestone review:** when a milestone's exit criteria are met, the team reviews it and the next epic is broken down into issues.
