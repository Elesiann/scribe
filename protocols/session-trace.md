# Scribe — Session Trace Protocol (v0.1)

**Status:** v0 internal contract for agent-session evidence.

Session Trace is the normalized record of a work session before it becomes a
`ContextBundle`. It captures what happened during execution: goals, decisions,
attempts, commands, files, failures, current state, and next steps.

It is not a transcript dump. It is a compact operational trace.

## Pipeline position

```
agent session / transcript / hook output
              │
              ▼
SessionTrace
              │
              ▼
ContextBundle evidence[] + items[]
              │
              ▼
preset renderer
```

## When to use

Use `session_trace` when the artifact depends on work that happened inside an
agentic session, especially:

- `handoff` — continuation context for the next operator
- `daily` — what was attempted, finished, blocked, or left open today
- `adr` — decisions and alternatives that emerged during the session
- `retro` — attempts, failures, and lessons from execution
- `loose-ends` — explicit parked or blocked signals from the session

Do not use it for external evidence by itself. A trace can say "we looked up the
docs", but the external doc is a separate source if its content supports a
claim.

## Top-level shape

```json
{
  "version": "0.1",
  "session": {},
  "objective": {},
  "timeline": [],
  "decisions": [],
  "attempts": [],
  "artifacts": [],
  "state": {},
  "open_threads": [],
  "next_actions": [],
  "provenance": {},
  "meta": {}
}
```

## Fields

### `version`

Schema version string. Use `"0.1"` in MVP.

### `session`

Identity and boundaries of the session.

```json
{
  "id": "session-2026-05-31-001",
  "surface": "codex",
  "started_at": "2026-05-31T14:00:00-03:00",
  "ended_at": "2026-05-31T15:10:00-03:00",
  "repo": { "name": "scribe", "path": "/home/gio/scribe", "branch": "main" }
}
```

Required: `id` when available. If the surface has no stable session id, derive
one from timestamp + workspace.

### `objective`

The work the session was trying to accomplish.

```json
{
  "summary": "Add a session-trace protocol to Scribe",
  "source": "user_prompt",
  "confidence": "high"
}
```

Required: `summary`.

### `timeline`

Ordered milestones, not every message or command.

```json
[
  {
    "id": "event-001",
    "timestamp": "2026-05-31T14:04:00-03:00",
    "kind": "user_request",
    "summary": "User accepted session-trace and rejected other ECC-inspired ideas"
  },
  {
    "id": "event-002",
    "kind": "inspection",
    "summary": "Read Scribe protocols and handoff preset before editing"
  }
]
```

Allowed `kind`:
`user_request | inspection | implementation | command | test | review | pause | correction | handoff_note`.

Rules:

- Include only events that change understanding, state, or next steps.
- Keep command output summarized. The raw transcript remains outside the trace.
- Preserve user corrections; they often explain why the final path changed.

### `decisions`

Decisions made during the session.

```json
[
  {
    "id": "decision-001",
    "summary": "Implement only session-trace, not the broader ECC-inspired set",
    "rationale": "User saw value only in session-trace",
    "status": "accepted",
    "evidence": ["event-001"]
  }
]
```

Allowed `status`: `proposed | accepted | rejected | superseded`.

### `attempts`

Approaches tried, whether successful or not.

```json
[
  {
    "id": "attempt-001",
    "summary": "Inspect current protocols before adding a new one",
    "result": "success",
    "detail": "Confirmed ContextBundle is the right downstream contract",
    "evidence": ["event-002"]
  }
]
```

Allowed `result`: `success | failed | partial | skipped`.

Use `attempts` to prevent the next operator from repeating work.

### `artifacts`

Files, docs, branches, PRs, issues, or generated outputs touched or produced.

```json
[
  {
    "id": "artifact-001",
    "kind": "file",
    "path": "protocols/session-trace.md",
    "role": "new protocol"
  }
]
```

Allowed `kind`: `file | branch | commit | pr | issue | doc | command_output | generated_artifact | external_link`.

### `state`

Where the work stopped.

```json
{
  "status": "in_progress",
  "summary": "Protocol added; docs still need wiring",
  "branch": "main",
  "modified_files": ["protocols/session-trace.md"],
  "known_blockers": []
}
```

Allowed `status`: `not_started | in_progress | blocked | review_ready | complete | parked`.

### `open_threads`

Questions, risks, or unresolved decisions.

```json
[
  {
    "id": "open-001",
    "type": "preference",
    "summary": "Whether other ECC-inspired ideas are worth adopting remains undecided"
  }
]
```

Allowed `type`: `question | risk | blocker | preference | follow_up`.

### `next_actions`

Concrete continuation steps.

```json
[
  {
    "id": "next-001",
    "summary": "Wire session_trace into ContextBundle source kinds",
    "priority": "now"
  }
]
```

Allowed `priority`: `now | next | later`.

### `provenance`

Where the trace came from.

```json
{
  "source_id": "session_trace",
  "input_kind": "conversation",
  "captured_by": "agent",
  "raw_available": false
}
```

`source_id` should be `session_trace` when entering the `ContextBundle`.

### `meta`

Implementation notes.

```json
{
  "generated_at": "2026-05-31T15:10:00-03:00",
  "trace_confidence": "high",
  "redactions": []
}
```

## Mapping to ContextBundle

SessionTrace maps into the existing `ContextBundle` without introducing new
item types.

| SessionTrace field | ContextBundle target |
|---|---|
| `session.repo` | `entities.repos[]` |
| `objective.summary` | `summary.headline` or `summary.where_we_stopped` |
| `timeline[]` | `evidence[]` with `kind = session_event` |
| `decisions[]` | `items[]` with `type = decision` |
| `attempts[]` | `items[]` with `type = attempt` |
| `artifacts[]` | `items[]` with `type = artifact_ref` and/or `entities.files[]` |
| `state.known_blockers[]` | `items[]` with `type = blocker` |
| `open_threads[]` | `items[]` with `type = open_question`, `risk`, or `blocker` |
| `next_actions[]` | `items[]` with `type = next_action` |

Use evidence refs like `session_trace:event-002` or
`session_trace:decision-001`.

## Extraction rules

- Prefer user-stated intent over inferred intent.
- Preserve corrections and reversals; they are high-value handoff evidence.
- Record failed and skipped attempts when they shaped the final path.
- Do not include full command output unless the output itself is the artifact.
- Do not treat implementation activity as delivery unless the work reached a
  usable or reviewable state.
- Do not turn speculation into decisions. If it was not accepted, use
  `open_threads`.
- Keep the trace compact enough to read in under two minutes.

## Minimum useful trace

```json
{
  "version": "0.1",
  "session": { "id": "session-2026-05-31-001", "surface": "codex" },
  "objective": { "summary": "Implement session-trace support", "source": "user_prompt", "confidence": "high" },
  "timeline": [
    { "id": "event-001", "kind": "user_request", "summary": "User approved session-trace only" }
  ],
  "state": { "status": "in_progress", "summary": "Ready to add protocol wiring" },
  "next_actions": [
    { "id": "next-001", "summary": "Update ContextBundle docs to accept session_trace", "priority": "now" }
  ],
  "provenance": { "source_id": "session_trace", "input_kind": "conversation", "captured_by": "agent" }
}
```

## Non-goals (v0)

- Full transcript archival
- Agent telemetry format standardization
- Automatic memory writes
- Publishing artifacts at session end
- Replacing `ContextBundle`

SessionTrace is an evidence adapter boundary. The renderer still consumes the
`ContextBundle`.
