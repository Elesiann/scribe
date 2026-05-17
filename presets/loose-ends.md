# Preset: loose-ends

**Lens:** triage / hygiene snapshot.

**Question it answers:** "What work is stalled, drifting, or intentionally parked — and what decision is needed now?"

**Default audience:** self, tech lead, or operator trying to regain control of active work across connected systems.

## Contract

| Aspect | Value |
|---|---|
| Input | User asks what is stalled, drifting, parked, or needs a decision |
| Output | Markdown triage brief grouped by urgency tier, with rationale and recommended action per item |
| Scope | Default: current user across connected work-tracking systems over a recent period |
| Destination | Inline (default), file, or workspace-destination suggestion |

## Required inputs

Ask only if not clear from the prompt:

- **period** — lookback window for stale/drift judgment
  - Default: last 14 days
  - Accepts: `"last 7 days"`, `"since monday"`, `"this sprint"`, `"2026-02-15..2026-02-22"`

## Optional inputs

- **scope** — default: current user across connected work-tracking systems
  - Accepts: `"owner:any"`, a named project, workspace cluster, or initiative
- **stale_hint** — explicit threshold override from the user
  - Accepts: `"consider review items stale after 10 days"`, `"use 7d for drift"`
- **focus** — prioritize one initiative if named
- **destination** — `inline | file | workspace-destination-suggestion`

## Minimum viable signal

For `loose-ends` to render well:

- at least **one work-tracking source** must be `used`
- conversation-only is **invalid** for this preset

Work-tracking sources include:

- `git_local`
- `gh`
- connected work tracker / docs sources (e.g. `notion`, `atlassian`)
- custom MCPs or APIs that expose work objects (tasks, branches, builds, etc.)

Below this, Scribe asks:

> "I need at least one work-tracking source for a useful loose-ends pass. Do you want to connect a source, point me at a workspace/system, or switch to another preset?"

## Primary item types (ContextBundle)

`work_item | blocker | risk | next_action | open_question`

Secondary fields: per-item triage metadata:

- `triage_status`: `stale | drifting | parked`
- `why_flagged`
- `recommended_action`: `retake | reassign | close | defer`
- `flag_confidence`
- `last_activity_at`
- `owner`
- `system`

## Sources (priority)

| Source | Role |
|---|---|
| connected work trackers / custom MCP work trackers | task and status reality |
| code-hosting review signal / `git_local` | code movement, open reviews, branch momentum |
| conversation | explicit parked signals, scope overrides, user threshold hints |
| manual notes | explicit hold/defer context |

## Gathering (capability-gated)

Gather work objects, not prose. Normalize heterogeneous systems into comparable tracked items.

Good signals:

- open reviews or branches with low/no recent movement
- tasks marked active but showing weak execution signals
- items repeatedly referenced but never completed
- explicit hold / blocked / waiting / deferred markers
- status-vs-reality mismatch across systems

Rules:

- prefer **relative signals** over hardcoded thresholds
- if the user gives a threshold override, it wins
- if not, use contextual judgment ("materially out of pace with peers", "marked active but quiet", "status says active, evidence says otherwise")
- if the same item appears across multiple work-tracking sources, compare explicit fields before assigning a triage label:
  - `status`
  - `priority`
  - `owner`
  - `last_activity_at` / `updated_at`
- do not split output by source system
- do not emit passive backlog inventory

## Extraction (LLM pass 1)

Normalize evidence into tracked work objects with these minimum questions answered:

1. what is stuck?
2. why is it suspect?
3. what decision needs to happen now?

Per surfaced item, extract or derive:

- `summary` — item name in plain language
- `owner` — default current user if not otherwise scoped
- `system` — most relevant source system
- `last_activity_at` — if known
- `triage_status` — `stale | drifting | parked`
- `why_flagged` — short rationale grounded in evidence
- `recommended_action` — `retake | reassign | close | defer`
- `confidence` — confidence about the item facts
- `flag_confidence` — confidence about the triage label itself
- `evidence_refs` — canonical provenance
- `source_mismatch` — optional, when two sources disagree on explicit status/priority/owner/activity fields

### Triage status rules

- `stale` = clearly stopped relative to declared or contextual expectations; decision needed now
- `drifting` = still moving or recently touched, but losing momentum or showing mismatch
- `parked` = intentionally paused, explicitly signaled in evidence

`parked` is conservative:

- only emit `parked` when there is an explicit label, note, comment, field, or user statement
- otherwise prefer `drifting`
- if no parked items are detected, omit the entire `Parked` section

### Cross-source mismatch rule

If the same work item appears across multiple work-tracking sources and those sources disagree on explicit fields, do not silently pick one view and proceed as if nothing happened.

Allowed behavior:

- keep the item
- surface the disagreement in `why_flagged`
- lower `flag_confidence`
- prefer `drifting` over an overconfident `stale` or `parked` label when the mismatch itself is the strongest signal

Examples:

- `Why flagged: cross-source mismatch — one tracker says In Progress, but another still looks backlog-like and shows no execution signal`
- `Why flagged: status mismatch — one tracker says active, another still shows blocked`

Not allowed:

- silently flattening conflicting source truth into a confident single-state claim

### Confidence rules

Treat confidence as two layers:

- `confidence` = confidence in the item facts
- `flag_confidence` = confidence that the item deserves the triage label

Example:

- factual confidence: high — review item exists, open for 18 days, linked work item still active
- flag confidence: medium — "stale" depends on team cadence and comparable work

## Synthesis (LLM pass 2)

Tone: direct, compact, decision-oriented.

Rules:

- group by urgency tier, never by source system
- prefer 5 high-signal items over 30 low-signal items
- fail closed: if an item cannot justify suspicion + decision pressure, omit it
- each item must include why it was flagged, evidence refs, and suggested action
- if `weekly` surfaced unresolved WIP but did not investigate it, this preset may deepen it
- if one item clearly needs packaging for continuation, suggest `handoff`
- when cross-source mismatch is the main signal, say so explicitly instead of pretending the item is cleanly classified

## Output template

See [examples/loose-ends.good.md](../examples/loose-ends.good.md) for the canonical shape.

Skeleton:
```markdown
# Loose Ends — <period>

Scope: owner=<scope>, systems=<used work-tracking systems>

## Stale — decide now
- <item title>
  Why flagged: <why suspect>
  Suggested action: <retake | reassign | close | defer>
  Confidence: facts=<high|medium|low>, flag=<high|medium|low>
  Evidence: <refs>

## Drifting — watch
- <item title>
  Why flagged: <why suspect>
  Suggested action: <retake | reassign | close | defer>
  Confidence: facts=<...>, flag=<...>
  Evidence: <refs>

## Parked — intentional
- <item title>
  Why parked: <explicit signal>
  Suggested action: <retake | reassign | close | defer>
  Confidence: facts=<...>, flag=<...>
  Evidence: <refs>

## Recommended actions
- <top action 1>
- <top action 2>

---
Sources
- used: ...
```

## Degradation

| Situation | Behavior |
|---|---|
| Only one work-tracking source available | Render a narrower triage and lower `flag_confidence` when comparison is weak |
| User provides explicit stale threshold | Use it and say so in `why flagged` |
| Mixed signals, weak suspicion | Keep the item out rather than pad the report |
| Cross-source conflict on explicit status/priority/activity | Keep the item only if the mismatch itself is operationally relevant; surface it and lower `flag_confidence` |
| No parked signals | Omit `Parked` entirely |

## Do / Don't

**Do:**
- investigate across systems, not inside one silo
- make decision pressure explicit
- surface evidence and uncertainty honestly
- keep scope visible at the top
- suggest `handoff` when one item deserves continuation packaging

**Don't:**
- don't turn this into a backlog audit
- don't group by specific source systems
- don't invent intentionality; no explicit signal means not parked
- don't emit items that cannot answer what is stuck / why suspect / what decision now
- don't rewrite the weekly in triage language
- don't silently flatten conflicting source truth into a single confident state

## Validation checklist

- Does every surfaced item answer what is stuck, why it is suspect, and what decision is needed now?
- Is `parked` used only when there is explicit evidence of intentional pause?
- Are `why flagged`, `recommended_action`, `confidence`, and `evidence` present for every item?
- If multiple work-tracking sources touched the same item, were explicit conflicts surfaced instead of flattened?
- Is the output grouped by urgency tier rather than source system?
- Does the output match the loose-ends example shape and stay in one language?
