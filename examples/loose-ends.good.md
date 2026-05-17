# Loose Ends — Feb 09–20, 2026

Scope: owner=current_user, systems=git_local, gh, work_tracker, issue_tracker

## Stale — decide now

- ACME-187 streaming-writer ADR implementation
  Why flagged: marked high-priority in the tracker and referenced repeatedly across weekly notes and PR discussions as a gating item for XLSX import, but no implementation branch or PR exists yet and no owner is assigned.
  Suggested action: retake
  Confidence: facts=high, flag=high
  Evidence: issue_tracker:ACME-187, work_tracker:weekly-2026-02-18, conversation:streaming-writer-followup

- `feat/sync-conflict-replay` branch
  Why flagged: branch open for 17 days with no commits in the last 11; the related work item shipped a different path (linearized retries) but the branch was never closed or rebased. Status implies active work, evidence suggests it was superseded.
  Suggested action: close
  Confidence: facts=medium, flag=medium
  Evidence: git:branch:feat/sync-conflict-replay, conversation:sync-direction-pivot

- `README.md` drift in `tools/fixture-gen`
  Why flagged: the script was promoted from a one-off helper to a documented dev tool, but the local README still reads as a spike notes file. Small work, but blocks the tool from feeling supported.
  Suggested action: retake
  Confidence: facts=high, flag=medium
  Evidence: git:status:tools/fixture-gen, conversation:fixture-gen-promotion

## Drifting — watch

- Mobile initial-load reduction plan
  Why flagged: discussed in two recent updates and flagged externally to the Acme Retail client, but the current signal is plan acknowledgment rather than active execution. Momentum exists conceptually, not implementation-wise.
  Suggested action: reassign
  Confidence: facts=medium, flag=medium
  Evidence: conversation:initial-load-discussion, work_tracker:acme-retail-sprint-7

- Large-workspace fixture set
  Why flagged: agreed on in the retro as a recurring gap, but no task was opened and no one named it as their cycle work. Mentioned, not owned.
  Suggested action: retake
  Confidence: facts=high, flag=medium
  Evidence: conversation:retro-2026-02-20, work_tracker:none

## Parked — intentional

- XLSX importer
  Why parked: explicitly deferred until the streaming-writer ADR lands; decision recorded in the retro and reaffirmed in the sprint scope discussion. Not abandoned, blocked by a known dependency.
  Suggested action: defer
  Confidence: facts=high, flag=high
  Evidence: conversation:retro-2026-02-20, work_tracker:ACME-204

## Recommended actions

- Assign an owner to ACME-187 (streaming-writer ADR) before it stays "high priority" without execution for another cycle; multiple downstream items depend on it.
- Close or explicitly park `feat/sync-conflict-replay`; an ambiguous branch is costing more than the work itself.
- Open the large-workspace fixture task this week and tie it to whoever picks up the next scale-related work.
- If the mobile initial-load plan stays ownerless by the next planning cycle, reassign it instead of carrying it as a background concern.

---
Sources
- used: conversation, git_local, work_tracker, issue_tracker, gh
