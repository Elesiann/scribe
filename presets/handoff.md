# Preset: handoff

**Lens:** continuity.

**Question it answers:** "How do I continue without rebuilding context?"

**Default audience:** next operator — human teammate, future session, or another agent.

## Contract

| Aspect | Value |
|---|---|
| Input | User asks for a package to hand off context |
| Output | Structured Markdown: context, objective, current state, decisions, attempts, blockers, next steps, **suggested resumption prompt** |
| Scope | Current session/resource/workstream (non-temporal fixed) |
| Destination | Inline (default), clipboard, or file |

## Required inputs

- **objective** — what is being done
  - Extractable from the active conversation or dominant resource name. Ask only if not inferable.

## Optional inputs

- **target_audience** — `next-session | teammate | external-agent` (default: `next-session`)
- **destination** — `inline | clipboard | file`

## Minimum viable signal

Requires:
- `objective` declared or inferable from session, resource, tracker, branch, or conversation
- At least 1 item in current state (session state, modified artifact, task/doc state, branch info, recent decision, or blocker)

Without these: *"What's the objective? I need to know what's being continued."*

## Primary item types (ContextBundle)

`decision | attempt | blocker | open_question | next_action | work_item`

Secondary fields: `entities.repos`, `entities.files`, `summary.where_we_stopped`.

## Source selection

Handoff is harness-agnostic. Use the sources that best explain how to resume the
selected workstream without rebuilding context.

| Source | Role |
|---|---|
| `session_trace` | compact record of the active work session — objective, decisions, attempts, state, next actions |
| `git_local` | current local code state when code artifacts are part of the handoff |
| `conversation` | decisions and attempts from the session |
| `filesystem` | files/resources in progress |
| connected trackers/docs | related work items or docs (optional) |

## Gathering (capability-gated)

### session_trace (if available)

Follow [../protocols/session-trace.md](../protocols/session-trace.md). Prefer a
session trace over re-reading the whole transcript when the surface provides one.

Captures: objective, important timeline events, decisions, attempts, artifacts,
current state, open threads, and concrete next actions.

### git (if available and relevant)

```bash
# current state
git rev-parse --abbrev-ref HEAD
git status --short
git log -5 --format='%h|%ai|%s' --date=iso

# in-progress branches
git branch --list --sort=-committerdate | head -10

# last merge point
git log --merges -1 --format='%h|%s'
```

Captures: current branch, modified files, recent commits, neighboring branches.

### conversation (if available and relevant)

Filter by:
- **Decisions taken:** "we defined X", "we chose Y", "accepted Z"
- **Attempts:** "tried X, didn't work", "tested Y and it did", "already tried W"
- **Current state:** "now stuck on", "I'm at K"
- **Pending:** "still need to", "next is", "tomorrow I continue"

### filesystem (if `used`)

List modified files via `git status`. Optionally `Read` key in-progress files for snippets.

### Connected tracker/docs sources (optional)

Tasks or docs linked to the current objective/resource (if the conversation mentions them or the resource name is parseable).

## Extraction (LLM pass 1)

- `decision` (0-N): decisions already made that **should not be re-discussed**
- `attempt` (0-N): attempted approaches (success or failure) — prevents the next operator from repeating
- `blocker` (0-N): what's currently stopping progress
- `open_question` (0-N): open points to decide
- `next_action` (0-N): concrete steps to move forward
- `work_item` (0-N): work in progress

## Synthesis (LLM pass 2)

Tone: action-oriented, pragmatic. Think of the next operator as a **capable stranger** — explain enough context for them to pick up work in 5 minutes.

## Output template

See [examples/handoff.good.md](../examples/handoff.good.md) for the canonical shape.

Skeleton:
```markdown
# Handoff — <objective>

## Context
<2-4 lines: what's being done, why, what the scope is>

## Objective
<1-2 lines, concrete>

## Current state
- **Workspace / resource:** <repo, task, doc, project, or session scope>
- **Current state:** <branch, task status, doc state, modified artifacts, or equivalent>
- **Relevant recent evidence:** <change/review/task/doc/session ref + 1-line>

## Decisions already made
- <decision 1> — **don't re-discuss**
- <decision 2>

## Attempts made
- <approach A> — <result/status>
- <approach B> — <result/status>

## Current blockers
- <what's stalling> (omit if empty)

## Open questions
- <point 1 to decide>

## Next steps
1. <concrete action>
2. <concrete action>

## Risks
- <if relevant> (omit if empty)

## Files/resources involved
- `<path/to/file>` — <role>
- `<resource>` — <current state if relevant>

## Links
- <PR, task, doc>

## Suggested resumption prompt

```
Resuming work on <resource>.

Objective: <objective>.

Decisions already made:
- <decision 1 summary>
- <decision 2 summary>

Current state: <state in 1 line>.
Next step: <action>.

<extra specific context if needed>
```

---
Sources
- used: ...
- missing: ...
```

## Degradation

| Situation | Behavior |
|---|---|
| No git | Omit branch/code state; use session, tracker, docs, manual notes, or conversation |
| No substantive session/resource state | "Tell me what you're working on" |
| Ambiguous objective | Ask which among detected top-3 candidates (session objective, resource, branch, feature, task) |
| Workstream has no recent movement | Generate a parking handoff only if current context is still worth preserving |

## Do / Don't

**Do:**
- **Final suggested prompt is the killer feature** — write it dense, ready to paste
- Omit empty sections (no blockers → drop section)
- Specifics: branch name when available, plus any relevant IDs, filenames, links, or work item references
- Action-oriented in "Next steps" — numbered, clear verb
- Package one selected work item cleanly; if triage is still unclear, suggest `loose-ends` first

**Don't:**
- Don't generate a handoff without an objective
- Don't repeat context that would be in a weekly (handoff is **now-focused**, not historical)
- Don't list everything that happened — only what the next operator needs
- Don't do ADR's job (handoff describes state, ADR documents a decision)
- Don't mix with daily (daily is short past; handoff is present + future)
- Don't do investigation ranking inside the handoff; `loose-ends` decides what is suspect, handoff packages continuation

## Validation checklist

- Is the objective explicit and specific?
- Does the handoff package one work stream cleanly instead of summarizing the whole week?
- Is the suggested resumption prompt dense, accurate, and ready to paste?
- Does the output match the handoff example shape and stay in one language?
