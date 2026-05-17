# Handoff — Settlement reconciliation engine MVP

> **Proof asset:** this artifact was generated from [`demo-evidence.md`](demo-evidence.md) applying the `handoff` preset. Same evidence, continuity lens. Compare with `from-same-evidence.weekly.md` and `.adr.md`.

## Context

The settlement reconciliation engine ties acquirer settlement files against the internal ledger and routes every discrepancy to a typed dead-letter table. The architecture is decided and the skeleton compiles; the matcher logic and its test fixtures are the remaining load-bearing work.

## Objective

Bring the reconciliation engine to a trustworthy MVP:
1. Canonical fixtures covering every discrepancy reason code
2. Matcher implemented and gated on those fixtures
3. Dead-letter table migration wired
4. One end-to-end run over a real acquirer settlement file

## Current state

- **Branch:** `feat/reconciliation-engine` (no new commits this session — design + skeleton only)
- **Workspace:** `services/reconciliation/{domain,adapters,jobs}` + `docs/specs/`
- **In place:**
  - `domain/ledger-read-model.ts` (fold stubbed, compiles)
  - `adapters/provider-adapter.interface.ts` + `acquirer-a`/`acquirer-b` reference adapters (stubs)
  - `jobs/reconciliation-batch.ts` (replayable cursor scaffold)
  - 3 specs (engine design, ledger read-model schema, settlement matching rules)
- **Not started:** matcher logic, canonical fixtures, dead-letter migration

## Decisions already made — don't re-discuss

- **Event-sourced reconciliation** (fold the ledger), not snapshot-diff
- **Idempotency key** = `(provider_id, external_ref, amount_cents)`, not a random UUID
- **Batch + replayable cursor**, not a live stream
- **Typed dead-letter table** for discrepancies (reason codes enumerated in spec 3 §7)
- **Provider-agnostic matcher** behind the adapter interface — acquirer logic never enters core
- **Snapshot-diff and live-stream were evaluated and rejected** — do not reopen

## Attempts made

- **Snapshot-diff reconciliation** — abandoned (no audit trail)
- **Live-stream matching** — abandoned (failure recovery = stream-position reconstruction)
- **Random-UUID idempotency** — rejected (doesn't dedupe acquirer retries)
- **Per-acquirer hardcoded matching** — rejected (every acquirer edits core)

## Current blockers

- **Canonical fixtures absent.** The matcher cannot be trusted or even meaningfully reviewed without one fixture per discrepancy reason code plus a happy path. This is the gating item.

## Open questions

- **Discrepancy review surface:** in-app dashboard or scheduled export from the dead-letter table? Default leaning: export first, dashboard later. Not blocking the matcher.

## Next steps

1. **Build canonical fixtures** — one per reason code (amount_mismatch, missing_in_ledger, missing_in_settlement, duplicate_external_ref, currency_mismatch) + happy path ← **start here**
2. **Implement the matcher** against the fixtures; gate merge on green fixtures
3. **Wire the dead-letter table migration** from the discrepancy taxonomy (typed reason-code column + replay flag)
4. **Define the reconciliation-lag SLO**; set batch cadence from it
5. **End-to-end run** over a representative acquirer settlement file via `acquirer-a` adapter

## Risks

- **Weak fixtures** → matcher mis-matches silently → dead-letter table looks clean while money doesn't tie out
- **Scope inflation** into an ops console before the matcher is stable — MVP is matcher + dead-letter table; review UI is deferred (cut list in spec 1 §9)

## Files/repos involved

- `services/reconciliation/domain/ledger-read-model.ts` — ledger fold (stub)
- `services/reconciliation/adapters/provider-adapter.interface.ts` — normalized settlement shape
- `services/reconciliation/adapters/{acquirer-a,acquirer-b}.adapter.ts` — reference adapters (stubs)
- `services/reconciliation/jobs/reconciliation-batch.ts` — replayable-cursor batch
- `docs/specs/reconciliation-engine-design.md`
- `docs/specs/ledger-read-model-schema.md`
- `docs/specs/settlement-matching-rules.md`

## Suggested resumption prompt

```
Resuming the settlement reconciliation engine MVP.

State: branch feat/reconciliation-engine. Architecture decided, skeleton compiles.
Specs in docs/specs/{reconciliation-engine-design,ledger-read-model-schema,
settlement-matching-rules}.md. Matcher logic + fixtures NOT started.

Locked decisions (don't re-discuss):
- Event-sourced reconciliation (fold the ledger), not snapshot-diff
- Idempotency key = (provider_id, external_ref, amount_cents)
- Batch + replayable cursor, not live stream
- Typed dead-letter table for discrepancies
- Provider-agnostic matcher behind the adapter interface

Immediate next step: build canonical fixtures, one per discrepancy reason code
(amount_mismatch, missing_in_ledger, missing_in_settlement,
duplicate_external_ref, currency_mismatch) + happy path. Then implement the
matcher gated on those fixtures, then wire the dead-letter migration.

Start with the fixtures following the discrepancy taxonomy in spec 3 §7.
```

---
Sources
- used: conversation, filesystem, git_local

Scope: reconciliation design session → implementation handoff
Generated: 2026-02-20T17:55:00Z
