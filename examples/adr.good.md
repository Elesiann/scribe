# ADR — Adopt content-addressable storage for note attachments

**Status:** accepted
**Date:** 2026-02-09

## Context

The notes app stores attachments (images, PDFs, small binary blobs) referenced from notes. The current scheme stores attachments under `{workspace_id}/{note_id}/{filename}` on object storage. Three problems became visible as workspaces grew:

1. **Duplication.** The same file pasted into 5 notes is stored 5 times. Large image workspaces grow disk usage well past what the user actually owns in unique content.
2. **Sync churn.** Renaming a note rewrites every attachment path, so sync re-uploads the full payload on rename. Operationally this looks like the user "uploading their library" again whenever they restructure.
3. **Garbage collection is hard.** Detecting that an attachment is orphaned requires walking every note in the workspace. We've deferred GC twice already because the walk is too expensive on large workspaces.

A content-addressable scheme keys each blob by its hash and stores it once per workspace, with notes holding references.

## Decision

Attachments are stored content-addressably: `{workspace_id}/blobs/{sha256}`, with notes holding `attachment_ref` entries that resolve hash → blob.

Supporting structure:

- Upload path: hash the blob, check existence, skip upload if already present
- Reference counting on each `attachment_ref` insert/delete; GC drops blobs with refcount 0 in a periodic sweep
- Migration: lazy — old `{workspace_id}/{note_id}/{filename}` paths stay until the note is next edited, then re-saved under the new scheme

## Alternatives considered

- **Keep the per-note path scheme.** Rejected — the duplication and sync-churn problems compound with workspace size and we already see them in real workspaces.
- **Per-user dedup (hash keyed only by `sha256`, shared across workspaces).** Rejected — leaks the existence of a blob across tenants; opens a side channel where uploading a private file could reveal that the same file exists in another workspace.
- **Eager migration sweep.** Rejected — requires a one-shot job over every workspace's attachments; deferred to lazy migration so we don't take a maintenance window.

## Consequences

- **Positive:**
  - Duplicate blobs disappear within each workspace; expected ~30% reduction on image-heavy workspaces based on a sample
  - Renames stop re-uploading; sync becomes metadata-only for the rename case
  - GC becomes a refcount sweep instead of a workspace walk
- **Negative:**
  - Upload path gains a hash + existence-check round trip; small files become slightly slower
  - The team owns the refcount correctness; an off-by-one in the reference logic could orphan blobs or delete live ones
  - Lazy migration means two schemes coexist for an unbounded period; reads must handle both
- **Neutral / open:**
  - Per-workspace dedup means a globally popular file is still stored once per workspace; that's the deliberate isolation tradeoff
  - GC sweep cadence is unset; weekly is the starting point, may need to tune

## Next steps

- Wire the upload path with hash + existence-check; gate behind a feature flag for staged rollout
- Implement refcount updates on `attachment_ref` insert/delete; cover with property tests
- Ship the GC sweep job behind the same flag; run dry-run mode for two weeks before enabling deletes
- Document the dual-scheme read path so the lazy migration is visible to anyone touching attachments

## Links

- Engine design notes: `docs/specs/attachment-storage.md`
- Refcount schema: `docs/specs/attachment-refcount.md`
- PR thread (design review): #341

---
Sources
- used: conversation, git_local, work_tracker

Scope: design discussion over 3 prior sessions, finalized 2026-02-09
Generated: 2026-02-09T19:30:00Z
