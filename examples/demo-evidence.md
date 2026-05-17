# Demo Evidence — Settlement reconciliation design session (2026-02-20)

Shared evidence base that feeds the 3 proof assets (`from-same-evidence.weekly.md`, `.adr.md`, `.handoff.md`).

**Purpose:** demonstrate Scribe's core thesis — **same evidence, different preset, different artifact**. The bundle below generates 3 distinct artifacts without changing the input, only the preset.

**Period:** 2026-02-20 (design session, ~3h)

In production, the ContextBundle is assembled automatically by evidence adapters. Here we represent the bundle as structured markdown for human legibility and so the reader can cross-reference raw → items → preset outputs.

---

## Surface

```yaml
conversation: true
bash: true
filesystem: true
git: true        # feature branch, no new commits in this session
connectors: true # tracker not consulted in this session
gh: true
```

## Sources status

- `conversation` — **used** (design session, 16+ decisional turns)
- `filesystem` — **used** (docs/specs + service skeleton)
- `git_local` — **used** (branch `feat/reconciliation-engine`, no new commits this session)
- `notion` — **missing** (not consulted; not relevant to a design session)
- `atlassian` — **missing** (idem)

## Raw evidence

### Filesystem / artifact creations (timestamped)

```
13:55Z  mkdir services/reconciliation/{domain,adapters,jobs}
13:58Z  domain/ledger-read-model.ts (skeleton)
14:10Z  docs/specs/reconciliation-engine-design.md
14:25Z  docs/specs/ledger-read-model-schema.md
14:25Z  docs/specs/settlement-matching-rules.md
15:20Z  Reconcile the matching rules against the ledger schema
          (idempotency key derivation, discrepancy taxonomy)
16:40Z  adapters/provider-adapter.interface.ts
17:05Z  adapters/{acquirer-a,acquirer-b}.adapter.ts (reference impls)
17:20Z  jobs/reconciliation-batch.ts (replayable cursor)
```

### Conversation highlights (decisional/blocker/intent signals)

- "if we snapshot-diff we lose the audit trail" → push toward event-sourced reconciliation
- "two retries from the same acquirer must not double-settle" → idempotency key derivation
- "a live stream makes failure recovery a nightmare" → batch with replayable cursor
- "a discrepancy that's only in the logs is invisible to ops" → typed dead-letter table
- "we cannot touch core every time a new acquirer shows up" → adapter interface
- "the matcher is provider-agnostic; the adapter normalizes" → architecture separation
- "fixtures first, or we will accept false matches" → test-fixture priority
- "don't build the ops console yet, build the matcher" → scope boundary confirmed

### Files referenced

- `docs/specs/reconciliation-engine-design.md`
- `docs/specs/ledger-read-model-schema.md`
- `docs/specs/settlement-matching-rules.md`
- `services/reconciliation/domain/ledger-read-model.ts`
- `services/reconciliation/adapters/provider-adapter.interface.ts`
- `services/reconciliation/jobs/reconciliation-batch.ts`

---

## Items (normalized operational units)

### Decisions

```yaml
decision-001:
  type: decision
  summary: "Reconciliation is event-sourced from the ledger, not snapshot-diffed"
  detail: "Rebuild settlement state by folding ledger events, not by diffing two point-in-time snapshots. Preserves a replayable audit trail and makes discrepancies explainable."
  evidence_refs: [conversation:audit-trail, spec1:§2]
  confidence: high
  tags: [architecture, auditability]

decision-002:
  type: decision
  summary: "Idempotency key derived from (provider_id, external_ref, amount_cents)"
  detail: "A deterministic key from the settlement tuple, not a random UUID. Acquirer retries of the same settlement collapse to one ledger effect; double-settlement on retry becomes structurally impossible."
  evidence_refs: [conversation:double-settle, spec3:§4]
  confidence: high
  tags: [correctness, idempotency]

decision-003:
  type: decision
  summary: "Batch reconciliation with a replayable cursor, not a live stream"
  detail: "A scheduled batch over a durable cursor. Failure recovery is 'reset the cursor and replay', not 'reconstruct stream position'. Simpler operationally."
  evidence_refs: [conversation:failure-recovery, spec1:§5]
  confidence: high
  tags: [architecture, operability]

decision-004:
  type: decision
  summary: "Discrepancies written to a typed dead-letter table"
  detail: "Every unmatched or conflicting settlement lands in a dead-letter table with a typed reason code, not just a log line. Ops can query, triage, and replay."
  evidence_refs: [conversation:invisible-discrepancy, spec3:§7]
  confidence: high
  tags: [operability, transparency]

decision-005:
  type: decision
  summary: "Provider-agnostic matching via an adapter interface"
  detail: "Core matcher consumes a normalized settlement shape. Each acquirer ships an adapter that maps its file format into that shape. New acquirers add an adapter; core does not change."
  evidence_refs: [conversation:adapter-boundary, spec1:§3]
  confidence: high
  tags: [architecture, extensibility]
```

### Deliveries

```yaml
delivery-001:
  type: delivery
  summary: "Reconciliation service skeleton + ledger read model"
  detail: "services/reconciliation/{domain,adapters,jobs} created; ledger-read-model.ts stubs the fold over ledger events. Compiles, no logic yet."
  evidence_refs: [filesystem:reconciliation-dir]
  confidence: high

delivery-002:
  type: delivery
  summary: "3 design specs written in docs/specs/"
  detail: "Reconciliation engine design (640 lines), ledger read-model schema (410 lines), settlement matching rules (520 lines). Formal basis of the feature."
  evidence_refs: [filesystem:specs]
  confidence: high

delivery-003:
  type: delivery
  summary: "Adapter interface + 2 reference adapters"
  detail: "provider-adapter.interface.ts defines the normalized shape; acquirer-a and acquirer-b adapters implement it as references for future acquirers."
  evidence_refs: [filesystem:adapters]
  confidence: high

delivery-004:
  type: delivery
  summary: "Discrepancy taxonomy (typed reason codes)"
  detail: "Reason codes enumerated: amount_mismatch, missing_in_ledger, missing_in_settlement, duplicate_external_ref, currency_mismatch. Drives the dead-letter table schema."
  evidence_refs: [filesystem:specs, spec3:§7]
  confidence: high
```

### Attempts

```yaml
attempt-001:
  type: attempt
  summary: "Snapshot-diff reconciliation"
  detail: "First approach diffed two balance snapshots to find drift."
  outcome: "deprecated — no audit trail, discrepancies not explainable; replaced by event-sourced fold"
  evidence_refs: [conversation:audit-trail]

attempt-002:
  type: attempt
  summary: "Live-stream matching"
  detail: "Considered matching settlements as they stream in."
  outcome: "abandoned — failure recovery requires reconstructing stream position; batch+cursor preferred"
  evidence_refs: [conversation:failure-recovery]

attempt-003:
  type: attempt
  summary: "Random-UUID idempotency keys"
  detail: "Initial idempotency design used a generated UUID per ingest."
  outcome: "rejected — does not dedupe acquirer retries of the same settlement; double-settlement stays possible"
  evidence_refs: [conversation:double-settle]

attempt-004:
  type: attempt
  summary: "Per-acquirer hardcoded matching"
  detail: "Quick path: a matching function per acquirer in core."
  outcome: "rejected — every new acquirer edits core; doesn't scale, adapter interface chosen instead"
  evidence_refs: [conversation:adapter-boundary]
```

### Blockers

```yaml
blocker-001:
  type: blocker
  summary: "Canonical fixtures + reference adapters still incomplete"
  detail: "Matcher cannot be trusted without a fixture set covering each reason code. acquirer-a/b adapters are stubs; fixtures absent. These are the load-bearing test assets."
  severity: high
  evidence_refs: [filesystem:adapters]
```

### Open questions

```yaml
open-question-001:
  type: open_question
  summary: "Discrepancy review: in-app dashboard or export to the ops tool?"
  detail: "Dead-letter table is the source of truth. The review surface could be a small in-app dashboard or a scheduled export. Affects scope."
  resolution_hint: "default to export first; dashboard is a later increment"
```

### Next actions

```yaml
next-action-001:
  type: next_action
  summary: "Build canonical fixtures covering every reason code"
  detail: "One fixture per discrepancy reason + happy path. Gate the matcher on these."
  priority: high

next-action-002:
  type: next_action
  summary: "Wire the dead-letter table migration"
  detail: "Schema from the discrepancy taxonomy; typed reason-code column + replay flag."

next-action-003:
  type: next_action
  summary: "Define the reconciliation-lag SLO"
  detail: "Maximum acceptable delay between settlement receipt and reconciliation. Drives batch cadence."

next-action-004:
  type: next_action
  summary: "Empirical test on a real settlement file"
  detail: "Run the batch over a representative acquirer settlement file end-to-end."
```

### Risks

```yaml
risk-001:
  type: risk
  summary: "Weak fixtures → matcher accepts false positives"
  detail: "If fixtures don't cover each reason code, the matcher can silently mis-match and the dead-letter table looks clean while money doesn't tie out."
  severity: high

risk-002:
  type: risk
  summary: "Scope inflating into a full ops console before the core matcher is stable"
  detail: "Review UI is tempting; the matcher + dead-letter table are the MVP. Cut list is explicit in spec 1 §9."
  severity: medium
```

---

## Summary

- **Headline:** Event-sourced settlement reconciliation locked; service skeleton + 3 specs + adapter interface + 2 reference adapters created in one session.
- **Themes:** reconciliation architecture, idempotency & correctness, adapter extensibility, discrepancy operability, implementation velocity.
- **Dominant risks:** fixtures are load-bearing; scope may inflate into an ops console.
- **Where we stopped:** skeleton + specs + adapter interface complete; fixtures + reference adapters + dead-letter migration still to do.

## Meta

- `generated_at`: 2026-02-20T17:30:00Z
- `bundle_confidence`: high
- `notes`: "Worked example — a payments-service settlement reconciliation design session. Input for the 3 from-same-evidence.*.md proof assets."
