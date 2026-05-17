# Preset: client-update

**Lens:** external-facing communication.

**Question it answers:** "What does someone outside day-to-day execution need to know?"

**Default audience:** client, stakeholder, leadership, or adjacent team that needs progress without implementation detail.

## Contract

| Aspect | Value |
|---|---|
| Input | User asks for a stakeholder-facing progress update |
| Output | Plain-language Markdown with snapshot, progress, impact, risks / asks, next steps |
| Scope | Usually a week or milestone window |
| Destination | Inline (default), file, email draft, or workspace-destination suggestion |

## Required inputs

Ask only if not clear from the prompt:

- **period** — time window
  - Default: last 7 days
  - Accepts: `"this week"`, `"since last update"`, `"this sprint"`, `"2026-02-15..2026-02-20"`

## Optional inputs

- **audience_hint** — `client | leadership | cross-functional | partner` (default: `client`)
- **destination** — `inline | file | workspace-destination-suggestion`
- **focus** — emphasize one initiative or project if the user names it

## Minimum viable signal

For `client-update` to render well, at least **one** of these must hold:
- `delivery` or `work_item` evidence exists for the period
- `decision` + `next_action` exist and are relevant to the audience
- `conversation` contains explicit stakeholder framing ("need to explain this to the client", "leadership update")

Below this, Scribe asks:

> "I don't have enough externally meaningful signal yet. Do you want a short progress note based on what we have, or should I wait for more evidence?"

## Primary item types (ContextBundle)

`delivery | decision | risk | blocker | next_action | work_item`

Secondary fields: `summary.headline`, `summary.themes`, `summary.where_we_stopped`.

## Sources (priority)

| Source | Role |
|---|---|
| weekly-like evidence sources | source material for recent progress |
| conversation | audience framing, impact wording, asks |
| connected trackers/docs | roadmap, milestone, or stakeholder-facing context |
| `git_local` / code-hosting review signal | supporting evidence only; never the output's framing |

## Gathering (capability-gated)

Reuse the same evidence strategy as `weekly`, then apply a stricter communication filter:

- prefer outcomes over implementation steps
- translate technical work into stakeholder-relevant impact
- keep blockers as risks or asks, not engineering complaints

Useful evidence patterns:

- merges, shipped features, or delivered milestones
- product or workflow changes that alter user/stakeholder experience
- explicit blockers that require expectation-setting
- concrete next steps with timeline direction

## Extraction (LLM pass 1)

Extract into `items[]`:

- `delivery` — externally meaningful progress
- `decision` — decisions that materially affect timing, scope, or approach
- `risk` — blockers or uncertainties the audience should know about
- `next_action` — what happens next, in plain terms
- `work_item` — supporting context when a delivery is not yet complete

Also derive:

- `stakeholder_impact` — what changed for the audience
- `asks` — explicit dependencies, approvals, or expectation-setting needs

## Synthesis (LLM pass 2)

Tone: clear, calm, low-jargon, stakeholder-safe.

Rules:

- translate technical progress into user or stakeholder impact
- avoid commit-by-commit narration
- mention risks without sounding alarmist
- if there is an ask, make it explicit
- assume the reader does not live in the codebase or in the implementation details

## Output template

See [examples/client-update.good.md](../examples/client-update.good.md) for the canonical shape.

Skeleton:
```markdown
# Client Update — <period or milestone>

## Snapshot
<2-4 lines: where things stand in plain language>

## Progress since last update
- <outcome 1>
- <outcome 2>

## What this changes for you
- <impact 1>
- <impact 2>

## Risks / asks
- <risk or ask> (omit if empty)

## Next steps
- <next step 1>
- <next step 2>

## Links
- <optional doc / milestone reference>

---
Sources
- used: ...
```

## Degradation

| Situation | Behavior |
|---|---|
| Only technical traces, no audience context | Render a concise update with explicit wording assumptions |
| No period given | Default to last 7 days |
| No stakeholder-relevant progress | Offer a brief holding update instead of fabricating momentum |

## Do / Don't

**Do:**
- translate engineering work into impact
- surface risks and asks clearly
- keep specifics inline only when they help credibility
- keep it readable by someone outside implementation

**Don't:**
- don't dump raw change logs, low-level IDs, or internal branch trivia
- don't sound like a weekly with fewer details
- don't bury the ask in the middle of the text
- don't over-promise certainty if the work is still exploratory

## Validation checklist

- Is the update understandable by someone outside day-to-day execution?
- Does it translate technical work into stakeholder impact?
- Are risks and asks explicit without sounding alarmist?
- Does the output match the client-update example shape and stay in one language?
