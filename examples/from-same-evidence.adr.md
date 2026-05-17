# ADR — Event-sourced settlement reconciliation over snapshot-diff

> **Proof asset:** this artifact was generated from [`demo-evidence.md`](demo-evidence.md) applying the `adr` preset. Same evidence, different lens. Compare with `from-same-evidence.weekly.md` and `.handoff.md`.

**Status:** accepted
**Date:** 2026-02-20

## Context

The settlement reconciliation engine needs to tie acquirer settlement files against the internal ledger and surface every discrepancy. During design, two framings were on the table:

1. **Snapshot-diff.** Take two point-in-time balance snapshots and diff them to find drift. Simple to describe, cheap to start.

2. **Event-sourced fold.** Rebuild settlement state by folding the ordered ledger events, and match acquirer settlements against that derived state.

The choice was forced by an operability requirement: when money does not tie out, ops must be able to explain *why* and *which* settlement caused it, and replay the decision. Snapshot-diff answers "the totals differ by X" but not "this settlement is the cause". The decision stayed open until the audit-trail requirement was made explicit in conversation.

## Decision

Reconciliation is **event-sourced**: settlement state is a fold over the ledger event log, and acquirer settlements are matched against that derived state.

Supporting structure:

- **Deterministic idempotency key** from `(provider_id, external_ref, amount_cents)` — acquirer retries of the same settlement collapse to one ledger effect
- **Batch with a replayable cursor** — failure recovery is "reset the cursor and replay", not "reconstruct stream position"
- **Provider-agnostic matcher behind an adapter interface** — acquirers ship adapters; core never changes per acquirer
- **Typed dead-letter table** — every unmatched/conflicting settlement lands with a reason code, queryable and replayable

## Alternatives considered

- **Snapshot-diff reconciliation.** Rejected — no audit trail; discrepancies are not explainable per-settlement, which fails the core operability requirement.

- **Live-stream matching.** Rejected as the execution model — failure recovery requires reconstructing stream position; a batch over a durable cursor is operationally simpler for the same correctness.

- **Random-UUID idempotency keys.** Rejected — a generated key per ingest does not dedupe acquirer retries of the same settlement, so double-settlement stays possible.

- **Per-acquirer hardcoded matching.** Rejected — every new acquirer would edit core; the adapter interface keeps core stable and isolates acquirer-specific parsing.

## Consequences

- **Positive:**
  - Discrepancies are explainable and replayable per-settlement — the operability requirement is met by construction
  - Double-settlement on retry is structurally impossible, not a runtime guard
  - New acquirers are an adapter, not a core change — extensibility without regression risk
  - The dead-letter table gives ops a queryable surface instead of log archaeology

- **Negative:**
  - Folding the ledger is more expensive than a snapshot diff; batch cadence must respect the reconciliation-lag SLO
  - The team owns the fold's correctness; the ledger event contract must stay stable
  - Fixtures become load-bearing — without coverage per reason code, the matcher can mis-match silently

- **Neutral / open:**
  - The discrepancy review surface (in-app dashboard vs. scheduled export) is deferred; the dead-letter table is the source of truth either way

## Next steps

- Build canonical fixtures covering every discrepancy reason code; gate the matcher on them
- Wire the dead-letter table migration from the discrepancy taxonomy
- Define the reconciliation-lag SLO so batch cadence is a measured choice, not a guess
- Keep the matcher provider-agnostic; resist acquirer-specific logic leaking into core

## Links

- Engine design spec: `docs/specs/reconciliation-engine-design.md`
- Ledger read-model schema: `docs/specs/ledger-read-model-schema.md`
- Settlement matching rules: `docs/specs/settlement-matching-rules.md`
- Service skeleton: `services/reconciliation/`

---
Sources
- used: conversation, filesystem, git_local

Scope: active design session (16+ decisional turns, 3 specs, service skeleton)
Generated: 2026-02-20T17:50:00Z
