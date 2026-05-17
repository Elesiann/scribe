# Retro — 2026-02-15 → 2026-02-20

## What went well

- CSV import landed end-to-end in one cycle: parser, dry-run, UI surface, and the doc with a sample file all shipped together rather than across three sprints.
- Sequencing the two PRs (parser first, helpers extraction second) kept each diff reviewable and avoided a giant "refactor + feature" PR.
- The N+1 in the membership cache was caught and fixed in-cycle instead of becoming a tracked regression.

## What was painful

- The sync engine queue size had been fixed at a value nobody had revisited; the resulting flaky tests went un-diagnosed for months. That's low-grade noise we paid for.
- The XLSX importer kept getting pulled into scope and pushed back out; the cycle spent real attention deciding *not* to do it three times.
- The membership-cache regression only surfaced under a fixture-heavy test run. The fixtures we have don't exercise the multi-thousand member case yet.

## What surprised us

- The duplicate-detection logic had drifted between the importer and manual-merge paths more than expected — the extraction was simpler than the divergence audit.
- The streaming-writer ADR turned out to be load-bearing for more than just XLSX import; export tests are starting to lean on it too.

## Decisions / lessons

- Fixtures need to cover a "large workspace" shape; without it, scale regressions get caught late.
- Tunable defaults beat fixed values for anything in a hot path; the cost of making something tunable is small compared to the cost of debugging the symptom later.
- When the same logic exists in two places and one of them is being touched, extracting it now is cheaper than the next divergence.

## Keep doing

- Sequencing dependent PRs instead of stacking everything into one diff.
- Shipping the doc alongside the feature in the same cycle, not as a follow-up that never lands.

## Change next time

- Open a "large-workspace fixture" task at the start of the cycle, not after a regression is caught.
- Stop re-litigating scope decisions mid-cycle; if XLSX is out, it's out until the streaming-writer ADR lands.
- When a fixed value is touched for any reason, take the 10 minutes to make it tunable rather than setting a new fixed value.

## Open questions

- Is the streaming-writer ADR a single-cycle implementation or does it need to be split?
- What's the right cadence for the GC sweep once the content-addressable attachments work ships?

---
Sources
- used: conversation, work_tracker, gh
- missing: git_local, issue_tracker
