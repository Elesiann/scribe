# Preset: daily

**Lens:** short temporal (typically last 24h).

**Question it answers:** "What do I need to communicate today?"

**Default audience:** self / team (standup prep).

## Contract

| Aspect | Value |
|---|---|
| Input | User asks for daily recap (explicit or paraphrased) |
| Output | Short Markdown: summary, progress, blockers, next, risks |
| Scope | Short temporal window (default 24h) |
| Destination | Inline (default), file, clipboard |

## Required inputs

Ask only if not clear from the prompt:
- **period** — time window (default: `since yesterday`). Accepts: `"today"`, `"since yesterday"`, `"last 12h"`

## Optional inputs

- **destination** — `inline | file`
- **resource** — override for the current code workspace or connected work context

## Minimum viable signal

For `daily` to render quality:
- at least 1 `work_item` OR `blocker` OR `next_action` extracted
- OR active session context with progress signal

Below this, Scribe asks: *"No substantive signals in this period. Paste today's notes or should I skip?"*

## Primary item types (ContextBundle)

`work_item | blocker | next_action | risk`

Secondary fields: `summary.headline`, `summary.where_we_stopped`.

## Source selection

Daily is harness-agnostic. Use the sources that best prove what changed or
needs communication for the selected day.

| Source | Role |
|---|---|
| `session_trace` | progress, attempts, decisions, blockers, and current state from the active work session |
| `git_local` | local implementation evidence when code work is in scope |
| `conversation` | active session signals |
| connected work tracker | tasks worked on today (optional enrichment) |
| connected issue tracker | issues transitioned today (optional) |

Do not require git for a daily unless the requested scope is code work. A daily
can be valid from tracker updates, session traces, manual notes, docs, or
conversation when they contain concrete work signals.

## Gathering (capability-gated)

### session_trace (if available)

Follow [../protocols/session-trace.md](../protocols/session-trace.md). Use it to
capture session-level progress without replaying the full transcript.

### git (if available and relevant)

```bash
git log --since "1 day ago" --author="$(git config user.email)" \
  --format='%h|%ai|%s' --date=iso | head -30

git status --short
git rev-parse --abbrev-ref HEAD
```

**Cap:** 30 commits.

### Connected work tracker (example: Notion)

`mcp__notion__notion-search` with filter `created_date_range.start_date` = today.
Follow-up with `notion-fetch` if status extraction is needed.

**Cap:** 10 items.

### Connected issue tracker (example: Atlassian)

JQL:
```
assignee = currentUser() AND updated >= -1d ORDER BY updated DESC
```

**Cap:** 10 items.

### conversation (if available and relevant)

Filter active session for:
- Progress markers ("did X", "finished Y", "deployed Z")
- Blockers ("stuck on", "waiting on", "expired API token")
- Next markers ("will do", "tomorrow", "next is")

## Extraction (LLM pass 1)

Produce in `items[]` of ContextBundle:

- `work_item`: each completed or materially advanced work thread → 1 synthesized item (cluster related evidence; do not emit 1 item per commit/task)
- `blocker`: stalled dependencies, active failures
- `next_action`: concrete planned steps
- `risk`: tight deadlines, fragile dependencies, possible regressions

## Synthesis (LLM pass 2)

- `summary.headline`: 1-2 lines of daily context
- Progress as bullets focused on **outcome** ("closed the CSV import path", not "made 5 commits")
- Blockers explicit — omit section if empty
- Next as concrete bullets (action, not wish)
- Risks only if something non-trivial
- Tone: plain engineering English, short, standup-friendly

## Output template

See [examples/daily.good.md](../examples/daily.good.md) for the canonical shape.

Skeleton:
```markdown
# Daily — <DD-MM-YYYY>

<summary in 1-2 lines>

## Progress
- <outcome 1>
- <outcome 2>

## Blockers
- <item> (omit section if empty)

## Next steps
- <concrete action>

## Risks
- <item> (omit section if empty)

---
Sources
- used: ...
- missing: ...
```

## Degradation

| Situation | Behavior |
|---|---|
| requested source is unavailable | Ask before fallback, following explicit source requirement rules |
| current workspace has no local code history, but tracker/session/manual evidence is strong | Proceed from those sources; do not treat missing git as degraded by itself |
| only weak intent-level conversation exists | Ask for notes |
| No activity in period | Don't generate daily — report: "Nothing detected today. Skip?" |

## Do / Don't

**Do:**
- Short — standup is ~2min read aloud
- Outcome, not counts
- Cite work item IDs or links inline when relevant
- Omit empty sections (not "Blockers: none")

**Don't:**
- Don't list raw commits
- Don't mix with weekly (short scope, short output)
- Don't inflate with low-signal meetings/emails; include them only when they changed work state, decisions, blockers, or next steps

## Validation checklist

- Is the output short enough to read aloud in a standup?
- Are progress items outcome-oriented rather than raw activity?
- Are blockers and next steps concrete, with empty sections omitted?
- Does the output match the daily example shape and stay in one language?
