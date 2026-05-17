# Preset: retro

**Lens:** retrospective learning.

**Question it answers:** "What did we learn, and what should change after this?"

**Default audience:** team, self, or delivery group reflecting on a week, sprint, or focused work stream.

## Contract

| Aspect | Value |
|---|---|
| Input | User asks for a retrospective or lessons-learned artifact |
| Output | Markdown with wins, pain points, surprises, lessons, keep/change actions |
| Scope | A period, sprint, milestone, or focused work stream |
| Destination | Inline (default), file, workspace-destination suggestion |

## Required inputs

Ask only if not clear from the prompt:

- **period** — time window or scope
  - Default: last 7 days
  - Accepts: `"this sprint"`, `"this week"`, `"incident follow-up"`, `"2026-02-15..2026-02-20"`

## Optional inputs

- **focus** — specific initiative, workspace cluster, or incident
- **destination** — `inline | file | workspace-destination-suggestion`

## Minimum viable signal

For `retro` to render well, at least **one** of these must hold:
- both a positive outcome and a pain point are visible in the evidence
- `attempt`, `blocker`, or `risk` items exist alongside deliveries or decisions
- `conversation` contains explicit reflective language ("what worked", "what hurt", "next time")

Below this, Scribe asks:

> "I don't have enough contrast for a useful retro yet. Do you want a lightweight reflection based on the current evidence, or should we use `weekly` instead?"

## Primary item types (ContextBundle)

`delivery | attempt | blocker | risk | decision | next_action | open_question | work_item`

Secondary fields: `summary.themes`, `summary.where_we_stopped`.

## Sources (priority)

| Source | Role |
|---|---|
| weekly-like evidence sources | raw material for the period |
| conversation | lessons, frustrations, surprises, intent |
| connected trackers/docs | explicit tasks, delays, completions |
| `git_local` / code-hosting review signal | supporting detail only |

## Gathering (capability-gated)

Reuse the same base evidence as `weekly`, then extract reflection-oriented signal:

- repeated retries or rework
- blockers that consumed more energy than expected
- decisions that improved throughput or clarity
- surprising outcomes, positive or negative
- actions worth keeping or changing next time

## Extraction (LLM pass 1)

Extract into `items[]`:

- `delivery` — what genuinely went well
- `attempt` — what was tried and how it turned out
- `blocker` — pain points or friction
- `risk` — recurring fragility or unresolved cost
- `decision` — choices that changed the trajectory
- `next_action` — concrete process/product changes
- `open_question` — unresolved learnings or follow-ups

Also derive:

- `wins`
- `pain_points`
- `surprises`
- `lessons`
- `keep_doing`
- `change_next_time`

## Synthesis (LLM pass 2)

Tone: reflective, specific, non-defensive.

Rules:

- do not turn the retro into a second weekly
- avoid blame language
- prefer evidence-backed lessons over vague "communication could improve"
- each "change next time" item should be actionable
- keep the writing honest and compact

## Output template

See [examples/retro.good.md](../examples/retro.good.md) for the canonical shape.

Skeleton:
```markdown
# Retro — <period or scope>

## What went well
- <win 1>
- <win 2>

## What was painful
- <pain point 1>
- <pain point 2>

## What surprised us
- <surprise 1>

## Decisions / lessons
- <lesson 1>
- <lesson 2>

## Keep doing
- <keep 1>

## Change next time
- <change 1>
- <change 2>

## Open questions
- <question 1> (omit if empty)

---
Sources
- used: ...
```

## Degradation

| Situation | Behavior |
|---|---|
| Mostly positive progress, little friction | Produce a lighter retro with fewer sections, but keep lessons explicit |
| Mostly blockers, little delivery | Render a pain-heavy retro, but still extract actionable changes |
| Weak evidence | Ask for 2-3 bullets on what worked / what hurt |

## Do / Don't

**Do:**
- be specific about what improved and what slowed the team down
- convert observations into lessons and actions
- keep the retro blameless and evidence-based
- omit empty sections instead of filling with fluff

**Don't:**
- don't rewrite the weekly in different words
- don't moralize or assign blame
- don't invent lessons that are not supported by the evidence
- don't make "change next time" vague

## Validation checklist

- Does the retro contain both contrast and learning, not just recap?
- Are lessons and changes evidence-backed and actionable?
- Is the tone blameless and specific?
- Does the output match the retro example shape and stay in one language?
