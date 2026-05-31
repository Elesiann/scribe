# Weekly Recap — 2026-02-15 → 2026-02-20

Dense week closing the CSV import workstream and hardening the sync engine. Three releases of the desktop client and the duplicate-detection refactor merged. A membership-lookup N+1 surfaced under load testing and pulled focus for half a day.

## Highlights
- ✅ **CSV import path closed** — parser, dry-run, column-mapping UI and import doc shipped together
- ✅ **Duplicate-detection helpers extracted** — used by import + manual merge, no behavior change
- ✅ **Sync engine queue size tunable per workspace** — quiet fix that resolved two long-standing flaky tests
- ⚠ **Membership lookup N+1** — root-caused via a load test; local fix shipped, follow-up to expose the cache TTL as config

## By theme

### Import path closeout

CSV importer landed end-to-end: parser, dry-run preview, mapping UI, and the import doc with a sample file. PR #312 (parser + dry-run) and PR #318 (duplicate-detection extraction) were sequenced because the helpers were duplicated across the manual-merge code path. Extracting them first would have inflated the parser PR; sequencing it after kept each PR reviewable.

The XLSX importer remains explicitly out of scope for this cycle — backlog item to revisit after the streaming-writer ADR is implemented.

### Sync engine hardening

The per-workspace outbound buffer used a fixed queue size; under fixture-heavy tests it overflowed silently and ate retries. Made it tunable per workspace, with a sane default; two flaky tests went green.

### Membership lookup regression

A load-test run surfaced an N+1 in the membership cache lookup; the cache wasn't keyed by `(workspace_id, member_id)`, so the hot path was re-fetching. Local fix shipped (commit `ccc3333`); follow-up: expose the cache TTL via config so it can be tuned per deployment.

### Misc / housekeeping

Bumped the toolchain pin, deleted a dead worker script, refreshed the contributor guide.

## Decisions
- **Streaming writer for export pipeline** chosen over batched buffer — locked in ADR-014 (this session's discussion + spec1)
- **Membership cache key** is `(workspace_id, member_id)`, not `member_id` alone — fixes the N+1 by construction (commit `ccc3333`)
- **XLSX importer deferred** — only revisit after the streaming-writer ADR lands; trying it earlier would re-cross the same code paths

## Blockers / risks
- **Streaming-writer ADR not yet implemented** — fixtures and export tests already lean on it; risk grows the longer it sits
- **Membership cache TTL still hardcoded** — fix shipped, but the config exposure didn't ship this week

## Where we stopped
- **Current branch:** `feat/duplicate-detection-extract` (PR #318 open, awaiting review)
- **Open PR:** PR #318
- **Membership cache fix:** local fix shipped; config exposure pending in next PR

## Next steps
- Land PR #318 once the parser PR clears
- Open the PR exposing the membership cache TTL as config
- Start the streaming-writer ADR implementation; this is the gating item for XLSX import

---
Sources
- used: conversation, git_local, work_tracker, gh
- missing: issue_tracker

Period: 2026-02-15 → 2026-02-20 (last 7 days)
Generated: 2026-02-20T17:30:00Z
