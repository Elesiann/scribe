# Scribe — Source Ledger Protocol

Every artifact produced by Scribe ends with a **source ledger**. It is not optional.

## Why

The source ledger is what separates Scribe from generic summarizers. It communicates:

1. **Evidence provenance** — the reviewer / future-you / teammate knows where each claim came from
2. **What was missing** — limitations are explicit, not hidden
3. **Output confidence** — weak vs. strong sources signal how much weight the artifact carries
4. **Auditability** — if something looks wrong, it can be traced back

Without the ledger, the output is indistinguishable from "ChatGPT summarized this".

## Canonical format

```
Sources
- used: <comma-separated source ids>
- missing: <comma-separated source ids>
- denied: <comma-separated source ids>
- error: <comma-separated source ids>
```

Optionally followed by a line with period/scope and timestamp:

```
Period: 2026-02-15 → 2026-02-20 (last 7 days)
Generated: 2026-02-20T17:30:00Z
```

## Status vocabulary (fixed 4)

The ledger uses exactly these states:

| Status | Meaning |
|---|---|
| `used` | Source was probed and contributed evidence to the artifact |
| `missing` | Source doesn't exist / wasn't linked in this surface |
| `denied` | Source exists but access was denied (permission, auth) |
| `error` | Source exists and access was granted, but returned an error (500, timeout, malformed) |

Each source appears in **exactly one bucket**. Never two.

## Source IDs

Use the IDs from `sources[].id` in the ContextBundle. Common examples:

- `conversation` — active session context
- `session_trace` — compact normalized record of an agent work session
- `manual` — notes pasted by the user
- `git_local` — local code workspace detected
- `gh` — authenticated code-hosting CLI
- `notion` — connected workspace/doc tracker
- `atlassian` — connected issue tracker
- `google_drive` / `gmail` / `calendar` — Google connectors
- `<custom>` — custom MCPs (any project-specific MCP server, etc.)

## Examples

### All four statuses present

```
Sources
- used: conversation, git_local
- missing: notion
- denied: atlassian
- error: gh
```

### Rich run (everything used)

```
Sources
- used: conversation, git_local, notion, gh
```

(Empty buckets may be omitted.)

### Minimal run (only conversation)

```
Sources
- used: conversation
- missing: git_local, notion, atlassian, gh

⚠ Artifact generated with conversation context only. Review before sharing.
```

The `⚠` note appears **automatically** when **only** `conversation` is in `used` — signaling a "draft" vs. "evidence-backed document".

### ADR with explicit provenance

```
Sources
- used: conversation, git_local
- missing: notion

Scope: active session (14 turns, 2 files touched)
Generated: 2026-02-20T17:30:00Z
```

## Ledger rules

### Inclusion — absolute rule

**Ledger = what was actually probed/attempted.** Nothing more.

- ✅ **Include:** sources that were probed AND the gathering phase attempted to consume
- ❌ **NEVER include:** sources that were "skipped by choice" (e.g., "git + gh was enough, so I didn't probe notion")
- ❌ **NEVER include:** sources that exist but are irrelevant to the preset (e.g., `gmail` in a technical `adr`)

**Operational rule:** if you never issued a probe/gather call to that source, it **does not enter the ledger in any bucket** (not `missing`, not `used`, not anything).

**Anti-pattern that breaks the rule:**
```
# WRONG — notion and atlassian weren't probed, just skipped
missing: notion, atlassian (not probed — git + gh was enough)
```

Correct version: **omit notion/atlassian from the ledger entirely**. If they weren't attempted, they don't show up.

### Non-negotiables
- The ledger is mandatory in **every** artifact (including degraded mode)
- Counts are honest: if a source returned 0 items, it still counts as `used` (probed but empty) — or `missing` (doesn't exist). Choose based on the probe status, not the item count
- Never invent sources — the ledger only lists what was actually probed
- Never collapse into "multiple sources" — list by id

### Format discipline
- Bucket order: `used`, `missing`, `denied`, `error` (always in that order)
- Comma-separated within each bucket
- Omit empty buckets (don't render `- denied:` if no source is in that state)

## Substitution labeling

When a preset resolves a source substitution (see Source Substitution Rule in `capability-detection.md`), label it explicitly in the ledger:

```
Sources
- used: conversation (substituting for git — not available in this surface)
- missing: git_local, gh
```

The substitution note goes **inside the source entry**, not as a separate line. This keeps the ledger a single lookup-style block.

## Implementation note

The ledger is emitted by the **rendering phase** (Step 7 of the SKILL.md flow), not during synthesis. Rendering reads the capability manifest and per-source item counts from the ContextBundle and formats them deterministically — no LLM reasoning is needed for the ledger itself, only lookup.

This keeps the ledger reliable: even if synthesis drifts elsewhere in the artifact, the ledger reflects exactly what was collected. **Hardened by design.**
