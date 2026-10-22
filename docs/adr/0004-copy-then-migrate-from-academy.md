# 0004. Copy from Academy now, migrate Academy later

- Status: Proposed
- Date: 2026-10-16

## Context

The engine's API will change a lot while it is being extracted. Making Academy depend on it from day one would require coordinated pull requests in two repositories for every interface change.

## Decision

- Generic code and its tests are copied from `BitPolito/bitcoin-academy` into this repository and refactored here.
- Every porting pull request records the Academy commit it was copied from, in the form `Source: bitcoin-academy@<sha> <path>`.
- Academy is not modified until the engine reaches a tagged release (milestone E5).
- Milestone E6 then migrates Academy to `agentic_rag[qvac]` and removes the duplicated code.
- Fixes made in Academy in the meantime are ported deliberately. A `source/academy-sync` issue tracks them.

## Consequences

- The engine can iterate freely without breaking Academy.
- There are temporarily two copies of the RAG code.
- Drift has to be tracked until E6.
- E6 has to re-baseline Academy's evaluation to prove parity.
