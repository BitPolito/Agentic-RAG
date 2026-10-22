## What

<!-- The change, in a few bullet points. -->

## Why

<!-- The motivation. Link the spec section or ADR when relevant. -->

Closes #

## How verified

<!-- The commands you ran and their results. Include the issue's "How to verify" commands. -->

```
uv run ruff check . && uv run ruff format --check .
uv run mypy
uv run lint-imports
uv run pytest
```

## Eval before/after

<!-- REQUIRED for any change to ingestion, chunking, search, fusion, reranking or the evidence pack. Paste the arag-eval SUMMARY blocks. Otherwise write "Not applicable". -->

## Reviewer notes

<!-- What reviewers should look at closely, known limitations, follow-ups. For code ported from Academy, add: Source: bitcoin-academy@<sha> <path> -->

## Checklist

- [ ] Branch is named `<type>/<issue>-<short-description>`.
- [ ] Commits follow Conventional Commits with a `Refs #<issue>` footer.
- [ ] Every acceptance criterion of the issue is covered by a test or a command.
- [ ] Port and contract changes have an accepted ADR.
- [ ] Docs are updated when behaviour or conventions change.
