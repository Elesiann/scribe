# Weekly Recap — Settlement Reconciliation Design (2026-02-20)

> **Proof asset:** this artifact was generated from [`demo-evidence.md`](demo-evidence.md) applying the `weekly` preset. Compare with `from-same-evidence.adr.md` and `from-same-evidence.handoff.md` — **same evidence, three different artifacts**.

A single concentrated design session: the settlement reconciliation engine moved from open question to event-sourced architecture with an executable skeleton. From idea to compiling service in ~3 hours.

## Highlights
- ✅ **Reconciliation model locked:** event-sourced fold over the ledger, not snapshot-diff — auditability preserved
- ✅ **Double-settlement made structurally impossible:** idempotency key derived from `(provider_id, external_ref, amount_cents)`
- ✅ **3 specs written** in `docs/specs/`: engine design, ledger read-model schema, settlement matching rules
- ✅ **Executable skeleton** under `services/reconciliation/{domain,adapters,jobs}` — compiles, stubs the ledger fold
- ✅ **Adapter interface + 2 reference adapters** so new acquirers never touch core
- ✅ **Discrepancy taxonomy** defined (typed reason codes) driving the dead-letter table schema

## By theme

### Reconciliation architecture (3 decisions, 2 pivots) — ~40% of period

Core locked: **event-sourced reconciliation** over snapshot-diff, batch with a **replayable cursor** over a live stream. Two early approaches were dropped — snapshot-diff lost the audit trail, live-stream made failure recovery a reconstruction problem. The matcher consumes a normalized settlement shape; provider specifics live behind an adapter.

### Correctness & idempotency (1 decision) — ~20%

Idempotency key derived deterministically from the settlement tuple instead of a random UUID. Acquirer retries of the same settlement collapse to a single ledger effect; double-settlement on retry stops being a runtime concern and becomes structurally impossible.

### Extensibility & operability (2 decisions, 1 delivery) — ~25%

Provider-agnostic matching via `provider-adapter.interface.ts`; acquirer-a and acquirer-b shipped as reference adapters. Discrepancies route to a typed dead-letter table (reason codes: amount_mismatch, missing_in_ledger, missing_in_settlement, duplicate_external_ref, currency_mismatch) so ops can query and replay instead of grepping logs.

### Implementation velocity (4 deliveries, ~3h) — ~15%

Skeleton → specs → adapter interface → reference adapters in one session. The matcher logic and fixtures remain; the structure is in place and compiles.

## Decisions
- **Event-sourced reconciliation**, not snapshot-diff (conversation:audit-trail + spec 1 §2)
- **Deterministic idempotency key** from the settlement tuple (conversation:double-settle + spec 3 §4)
- **Batch + replayable cursor**, not live stream (conversation:failure-recovery + spec 1 §5)
- **Typed dead-letter table** for discrepancies (conversation:invisible-discrepancy + spec 3 §7)
- **Provider-agnostic matching via adapter interface** (conversation:adapter-boundary + spec 1 §3)

## Blockers / risks
- **Canonical fixtures + reference adapters incomplete** — load-bearing for trusting the matcher; immediate priority
- **Dead-letter migration not yet wired** — schema designed, migration pending

## Where we stopped
- Service skeleton compiles; ledger fold stubbed
- 3 specs complete; adapter interface + 2 reference adapters in place
- Fixtures absent; matcher logic not implemented
- Dead-letter table migration pending

## Next steps
1. Build canonical fixtures covering every discrepancy reason code ← immediate
2. Implement the matcher against fixtures (gate on them)
3. Wire the dead-letter table migration
4. Define the reconciliation-lag SLO and batch cadence
5. Empirical test over a real acquirer settlement file

---
Sources
- used: conversation, filesystem, git_local

Period: 2026-02-20 (design session, ~3h)
Generated: 2026-02-20T17:45:00Z
