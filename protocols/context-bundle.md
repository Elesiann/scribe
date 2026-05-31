# Scribe — ContextBundle Protocol (v0.1)

**Status:** v0 internal contract for MVP.
**Reference:** ContextBundle schema specification (v0.1).

ContextBundle is the neutral contract between **evidence acquisition** and **preset rendering**. It normalizes any combination of sources into a single shape that all presets can consume.

When the evidence is an agent work session, normalize it first as a
[`SessionTrace`](session-trace.md), then map the trace into this bundle.

## Pipeline position

```
interface input → PresetInvocation → evidence acquisition → ContextBundle → preset renderer → Markdown
```

The bundle does not know whether the invocation came from a skill, a CLI, or a future API.

## Top-level shape

```json
{
  "version": "0.1",
  "request": {},
  "surface": {},
  "sources": [],
  "entities": {},
  "raw": {},
  "evidence": [],
  "items": [],
  "summary": {},
  "meta": {}
}
```

## Fields

### `version`
Schema version string. Use `"0.1"` in MVP.

### `request` — the PresetInvocation

Compiled invocation contract — what the engine will execute. Interface-agnostic.

```json
{
  "preset": "weekly",
  "preset_resolution": {
    "mode": "explicit",
    "matched_from": "Scribe weekly"
  },
  "audience": "default",
  "source_policy": "auto",
  "source_requirements": {
    "mode": "explicit",
    "requested_sources": ["code_hosting"],
    "matched_from": "based on my code-hosting activity",
    "fallback_approved": false
  },
  "output_language": "en-US",
  "output_format": "markdown",
  "period": { "label": "this_week", "start": "2026-02-15", "end": "2026-02-20" },
  "objective": "prepare weekly update",
  "user_prompt": "Scribe, generate my weekly"
}
```

- `preset`: `daily | weekly | adr | handoff | client-update | retro | loose-ends`
- `preset_resolution.mode`: `explicit | inferred | clarified`
- `preset_resolution.matched_from`: the original wording that resolved the preset
- `preset_resolution.note`: optional — clarification record
- `audience`: fixed `"default"` in MVP
- `source_policy`: fixed `"auto"` in MVP
- `source_requirements.mode`: `auto | explicit`
- `source_requirements.requested_sources`: optional list of source ids explicitly required by the user
- `source_requirements.matched_from`: original wording that created the source requirement
- `source_requirements.fallback_approved`: boolean; false until the user explicitly allows weaker/different sources
- `output_language`: resolved artifact language, e.g. `en-US` or `pt-BR`
- `output_format`: fixed `"markdown"` in MVP
- `period.label`: optional — `today | this_week | last_sprint | custom`
- `period.start` / `period.end`: ISO dates, optional
- `objective`: optional — user intent
- `user_prompt`: original request string

Language rule:

- the same bundle renders one artifact in one language
- language is resolved before rendering and stored in `request.output_language`
- the renderer must not mix languages unless the user explicitly asked for a bilingual artifact

Source rule:

- if `source_requirements.mode` is `explicit`, the engine must probe the requested sources first
- if an explicitly requested source is unavailable and `fallback_approved` is false, gathering must pause and ask the user before substitution
- fallback only becomes valid after explicit user approval

### `surface`

Runtime capabilities as boolean flags — independent of branding.

```json
{
  "conversation": true,
  "session_trace": true,
  "connectors": true,
  "mcp": true,
  "filesystem": true,
  "bash": true,
  "git": true,
  "gh": false
}
```

### `sources`

Candidate sources with their final status.

```json
[
  { "id": "conversation", "kind": "conversation", "status": "used" },
  { "id": "session_trace", "kind": "session_trace", "status": "used", "label": "Current agent session" },
  { "id": "git_local",    "kind": "git",          "status": "used", "label": "Local code workspace" },
  { "id": "work_tracker", "kind": "connector",    "status": "missing", "details": "Not exposed in this surface" }
]
```

Required: `id`, `kind`, `status`.
Optional: `label`, `details`, `probe`, `error_code`.

**Allowed `status`** (fixed 4): `used | missing | denied | error`.

**Allowed `kind`:** `conversation | session_trace | manual | connector | mcp | filesystem | git | gh`.

### `entities`

Identifiable objects that can be referenced.

```json
{
  "repos":  [{ "name": "project-a", "path": "/path/to/project-a", "branch": "feature-x" }],
  "files":  [{ "path": "…", "role": "reference" }],
  "links":  [{ "label": "Decision doc", "url": "https://example.com/doc" }]
}
```

All fields are optional.

### `raw`

Source-specific evidence, minimally processed. Sparse by default.

```json
{
  "manual_notes": ["Keep engine-first framing"],
  "session_trace": {
    "objective": "Add session-trace support",
    "state": "Protocol added; docs still need wiring"
  },
  "conversation": { "highlights": ["Surface-aware is plumbing"] },
  "git": {
    "repo_root": "/path/to/project-a",
    "branch": "feature-x",
    "status_summary": "2 modified",
    "commit_summaries": [{ "sha": "abc1234", "message": "Add spike notes" }]
  }
}
```

Rule: `raw` is optional but strongly recommended. Never mirror large payloads verbatim — summarize.

### `evidence`

Normalized evidence references with canonical provenance.

```json
[
  {
    "id": "git:commit:abc1234",
    "source_id": "git_local",
    "kind": "commit",
    "title": "Add Scribe preset engine spec",
    "excerpt": "Defines PresetInvocation",
    "timestamp": "2026-02-20T14:10:00Z"
  }
]
```

Recommended: `id`, `source_id`, `kind`, `title`.
Optional: `excerpt`, `timestamp`, `url`, `path`, `meta`.

### `items` — normalized operational units

**The most important field.** Extracted from evidence, consumed directly by presets.

```json
[
  {
    "id": "decision-001",
    "type": "decision",
    "summary": "Scribe is a preset engine, not a summarizer",
    "detail": "Same evidence → multiple artifacts by changing only the preset",
    "confidence": "high",
    "evidence_refs": ["conversation:decision-001"],
    "tags": ["architecture", "positioning"]
  }
]
```

Required: `id`, `type`, `summary`.
Recommended: `detail`, `confidence`, `evidence_refs`, `tags`, `links`, `time_range`.

Optional metadata frequently used by triage-oriented presets:

- `owner`: current owner or assignee
- `system`: source system most associated with the item (`git`, `code_hosting`, `work_tracker`, `docs_connector`, custom MCP id)
- `last_activity_at`: last meaningful activity timestamp known for the item
- `triage_status`: `stale | drifting | parked`
- `why_flagged`: short rationale for why the item was surfaced
- `recommended_action`: `retake | reassign | close | defer`
- `flag_confidence`: confidence about the triage label itself; separate from item factual confidence
- `source_mismatch`: optional object for explicit cross-source disagreement, e.g. conflicting `status`, `priority`, `owner`, or `last_activity_at`

**Allowed `type`** (fixed 9 in MVP):
`work_item | decision | risk | blocker | next_action | artifact_ref | delivery | attempt | open_question`

**Allowed `confidence`:** `high | medium | low`.

**Provenance rule:** prefer `evidence_refs` for canonical provenance; `evidence` inline may serve as a human-readable shorthand.

Session trace refs use the `session_trace:<trace-id>` form, for example
`session_trace:event-002` or `session_trace:decision-001`.

### `summary`

Preset-agnostic rollup that helps rendering.

```json
{
  "headline": "Preset engine framing accepted, MVP scope tightened",
  "themes": ["preset engine", "runtime plumbing", "MVP scope"],
  "where_we_stopped": "Ready to execute capability spike"
}
```

All optional: `headline`, `themes`, `where_we_stopped`, `dominant_risks`.

### `meta`

Implementation metadata.

```json
{ "generated_at": "2026-02-20T16:20:00Z", "bundle_confidence": "medium", "notes": [] }
```

## Preset expectations over the bundle

Each preset consumes a specific subset of item types.

| Preset | Primary item types | Secondary fields |
|---|---|---|
| `daily` | `work_item`, `blocker`, `next_action`, `risk` | `summary.headline`, `summary.where_we_stopped` |
| `weekly` | `delivery`, `decision`, `risk`, `blocker`, `next_action`, `work_item` | `summary.themes`, `summary.where_we_stopped` |
| `adr` | `decision`, `risk`, `open_question`, `next_action` | `entities.links`, `raw.manual_notes` |
| `handoff` | `decision`, `attempt`, `blocker`, `open_question`, `next_action`, `work_item` | `entities.repos`, `entities.files`, `summary.where_we_stopped` |
| `client-update` | `delivery`, `decision`, `risk`, `next_action`, `work_item` | `summary.headline`, `summary.themes` |
| `retro` | `delivery`, `decision`, `risk`, `blocker`, `attempt`, `next_action` | `summary.themes`, `meta.notes` |
| `loose-ends` | `work_item`, `blocker`, `risk`, `next_action`, `open_question` | per-item triage metadata (`triage_status`, `why_flagged`, `recommended_action`, `flag_confidence`, `last_activity_at`, `owner`, `system`) |

Session traces are especially useful for `handoff`, `daily`, `adr`, and
`retro`, because they preserve decisions, attempts, corrections, current state,
and concrete continuation steps from the work session.

## Extraction rules

- **Collectors emit evidence, not prose.** Rendering stays in the renderer.
- Merge duplicate facts across sources into a single `item`
- Preserve the strongest evidence source list in `evidence_refs`
- Map `SessionTrace.decisions[]` to `decision`, `attempts[]` to `attempt`,
  `artifacts[]` to `artifact_ref`, `open_threads[]` to `open_question` / `risk`
  / `blocker`, and `next_actions[]` to `next_action`
- Prefer `decision` over generic `work_item` when an architectural choice is clear
- Prefer `blocker` over `risk` when the issue is actively halting progress
- Emit `next_action` only when the action is concrete (not "think about X")
- For `loose-ends`, fail closed: if the item cannot answer "what is stuck / why is it suspect / what decision is needed now", do not emit it
- For `loose-ends`, only emit `triage_status = parked` when an explicit parked signal exists in evidence; otherwise prefer `drifting`
- For `loose-ends`, if the same work item appears across multiple work-tracking sources with conflicting explicit fields, do not flatten the conflict silently; record the mismatch, surface it in `why_flagged`, and lower `flag_confidence`

## Renderer rules

- Render exactly **one** resolved preset contract per artifact (no mixing)
- The same bundle may legitimately feed multiple presets without changing the evidence
- Same bundle + same preset → structurally stable output (prose varies, structure doesn't)

## Minimal valid bundle (graceful degradation)

```json
{
  "version": "0.1",
  "request": {
    "preset": "handoff",
    "preset_resolution": { "mode": "inferred", "matched_from": "continue tomorrow" },
    "audience": "default",
    "source_policy": "auto",
    "source_requirements": { "mode": "auto", "requested_sources": [], "fallback_approved": true },
    "output_language": "en-US",
    "output_format": "markdown",
    "objective": "continue the work tomorrow"
  },
  "surface": { "conversation": true, "connectors": false, "mcp": false, "filesystem": false, "bash": false, "git": false, "gh": false },
  "sources": [{ "id": "conversation", "kind": "conversation", "status": "used" }],
  "entities": {},
  "raw": { "manual_notes": ["Need to continue Scribe MVP design tomorrow"] },
  "evidence": [{ "id": "conversation:note-001", "source_id": "conversation", "kind": "note", "title": "Continue design tomorrow" }],
  "items": [
    { "id": "next-001", "type": "next_action", "summary": "Resume from accepted preset-engine architecture", "evidence_refs": ["conversation:note-001"] }
  ],
  "summary": { "headline": "Preset engine accepted", "where_we_stopped": "Write initial implementation docs" },
  "meta": {}
}
```

## Out-of-scope (v0)

- Audience-specific rendering metadata
- Auto-publication metadata
- Per-team compliance/privacy policies
- Scoring models
- Long-lived artifact history
- Source-specific auth payloads

## Implementation notes

- Keep the schema JSON-serializable
- Prefer sparse objects — don't force empty arrays
- Don't require every collector to populate every field
- Treat this as a **stable internal contract** for the MVP
