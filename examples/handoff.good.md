# Handoff — Finish duplicate-detection helpers extraction

## Context

Duplicate-detection logic was copy-pasted across the CSV importer and the manual-merge UI. PR #312 (importer base) shipped with the importer keeping its inline copy; this branch extracts both call sites onto a shared module so the next importer (XLSX) can reuse it without re-introducing the divergence.

## Objective

Land PR #318 (`feat/duplicate-detection-extract`): extract the two duplicate-detection implementations into `lib/dedup/`, swap both call sites, and prove behavior parity via the existing fixture set.

## Current state

- **Branch:** `feat/duplicate-detection-extract`
- **Uncommitted modifications:** none (last commit `def5678` pushed)
- **Relevant recent commits:**
  - `def5678` — swap manual-merge call site to `lib/dedup/match.ts`
  - `aaa1111` — extract shared scoring function from importer
  - `bbb2222` — scaffold `lib/dedup/` with re-exports

## Decisions already made — don't re-discuss

- **Shared module lives at `lib/dedup/`**, not under the importer — manual-merge predates the importer and shouldn't be a subpackage of it
- **Scoring weights stay as constants in the shared module**, not config — the existing fixtures lock the current weights; changing them is a separate decision
- **Both call sites get the same module**, not parallel implementations behind a feature flag — the whole point is removing the divergence
- **No behavior change in this PR** — anything that isn't covered by the fixture parity check is out of scope

## Attempts made

- **Approach 1:** keep the importer's inline implementation and only refactor manual-merge — abandoned; doesn't solve the divergence for the next importer
- **Approach 2:** introduce a `DedupStrategy` interface with a default impl — over-engineered; PR feedback (locked decisions above) rejected it
- **Approach 3 (current):** plain module with named exports, both call sites swap to it — accepted

## Current blockers

- PR #312 (importer base) not yet merged; PR #318 stacks on top of it via `git rebase`. Local rebase clean, but reviewers can't see a clean diff until #312 lands.

## Open questions

- Should the XLSX importer task (currently in the backlog) be unblocked off this PR or off the streaming-writer ADR? Leaning toward streaming-writer first; this PR is necessary but not sufficient.

## Next steps

1. Watch PR #312 land; rebase #318 on `main` once it does
2. Re-run the fixture parity check (`pnpm test lib/dedup`) and confirm green on the rebased branch
3. Request review on #318; the only non-trivial reviewer concern will be the equivalence of the swapped call sites
4. Once merged, close the "extract duplicate-detection" tracker item and link the XLSX importer ticket as the next downstream consumer

## Risks

- Rebase of #318 on top of an updated #312 may surface a conflict in the importer call site (the same function is being touched on both branches)
- The fixture set is the only behavior contract; if a real-world duplicate pattern isn't covered, the swap could miss it silently — fixture coverage gap is the largest residual risk

## Files/resources involved

- `lib/dedup/match.ts` — extracted matcher (new)
- `lib/dedup/scoring.ts` — extracted scoring (new)
- `apps/web/manual-merge/dedup-panel.tsx` — call site swap
- `packages/importer-csv/src/duplicate-check.ts` — call site swap
- `lib/dedup/__fixtures__/` — parity fixture set

## Links

- PR #318 (open, awaiting #312) — https://example.com/acme-notes/pull/318
- PR #312 (importer base, awaiting review) — https://example.com/acme-notes/pull/312
- Backlog: XLSX importer ticket — ACME-204

## Suggested resumption prompt

```
Resuming the duplicate-detection helpers extraction.

State: branch feat/duplicate-detection-extract, last commit def5678 pushed.
PR #318 open, stacked on PR #312 (importer base, not yet merged). Both call
sites (manual-merge + CSV importer) already swapped to lib/dedup/; fixture
parity check green locally.

Locked decisions: shared module at lib/dedup/, scoring weights stay as
constants, no behavior change in this PR, no DedupStrategy interface.

Immediate next step:
1. Wait for PR #312 to merge
2. Rebase #318 on main; expect a possible conflict in the importer call site
3. Re-run `pnpm test lib/dedup` and confirm fixture parity
4. Request review on #318

Backlog: XLSX importer (ACME-204) is the next downstream consumer; it is
blocked by the streaming-writer ADR, not by this PR.
```

---
Sources
- used: conversation, git_local, gh
- missing: issue_tracker, work_tracker

Scope: session 2026-02-20 (~22:00 UTC), branch feat/duplicate-detection-extract
Generated: 2026-02-20T22:30:00Z
