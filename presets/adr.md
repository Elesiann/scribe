# Preset: adr

**Lens:** decisional.

**Question it answers:** "What decision needs to become memory?"

**Default audience:** engineering / future-you / team.

**Format:** MADR-inspired Markdown ADR, portable.

## Contract

| Aspect | Value |
|---|---|
| Input | User asks to record a technical/architectural decision |
| Output | Markdown ADR (title, status, context, decision, alternatives, consequences, next steps, links) |
| Scope | Discussed/made decision (non-temporal) |
| Destination | Inline (default) — can be saved to `docs/decisions/` or similar |

## Relationship to a living tracker

An ADR is not a tracker task. The two serve different lifecycles and should not duplicate each other.

| Aspect | Living tracker task | scribe adr |
|---|---|---|
| Destination | Task/issue in a tracker | Portable Markdown |
| Lifecycle | Evolves over time | Immutable document (committable to repo) |
| Audience | Tracker viewers | Repo readers, future devs |
| When to use | Decision becomes operational work | Decision becomes architectural memory |

**Rule:** they don't duplicate. If the decision needs to drive ongoing operational work, that belongs in a tracker. If it needs to become committable architectural memory, that's `scribe adr`. Using both at different moments is fine; just don't mirror one into the other.

## Required inputs

- **decision_subject** — what was decided
  - Extractable from session trace, conversation, linked docs, tracker discussion, or code context. Ask only if ambiguous among 2+ detected decisions.

## Optional inputs

- **status** — `proposed | accepted | deprecated | superseded` (default: `accepted` if the decision has already been made)
- **destination** — `inline | file | suggest-destination` (suggestion only, never auto-publishes)

## Minimum viable signal

Scribe requires **at least one clear or candidate decision**. If there isn't one:
> "I didn't detect a clear decision in the available evidence. What decision do you want to document?"

Never generate an ADR with a fabricated decision.

## Primary item types (ContextBundle)

`decision | risk | open_question | next_action`

Secondary fields: `entities.links`, `raw.manual_notes`.

## Source selection

ADR is evidence-led, not conversation-led. Use the source that best captures the
decision, rationale, alternatives, and consequences.

| Source | Role |
|---|---|
| `session_trace` | accepted/rejected decisions, rationale, attempts, and corrections from the work session |
| `conversation` | discussion, decision, rationale |
| `git_local` | contextualizes affected code or local changes (diff, files touched) |
| `filesystem` | read referenced files/resources |
| connected docs/tracker sources | linked docs/tasks (if mentioned) |

## Gathering (capability-gated)

### session_trace (if available)

Follow [../protocols/session-trace.md](../protocols/session-trace.md). Use
`decisions[]`, `attempts[]`, and correction events to identify the decision,
real alternatives, and rationale.

### conversation (if available and relevant)

Filter by semantic markers:
- **Explicit decision:** "going with X", "we chose Y", "rejected Z", "decided"
- **Rationale:** "because of", "to avoid", "since", "because"
- **Alternatives:** "considered A", "thought about B but", "alternative would be"
- **Consequences:** "this implies", "will need", "the cost is"

### git (if available and relevant)

```bash
git diff HEAD~5..HEAD --stat 2>/dev/null
git log --since "1 day ago" --format='%h|%s' | head -10
git status --short
```

Contextualizes the ADR with "code related to the decision".

### filesystem (if `used`)

If the conversation mentions specific files, a quick `Read` to extract relevant snippets for the `Context` section.

### Connected docs/tracker sources (optional)

If the conversation links pages/issues, a quick fetch for the `Links` section.

## Extraction (LLM pass 1)

- `decision` (mandatory, exactly 1 per ADR): the decision made, in 1 line
- `risk` (0-N): negative consequences or risks introduced
- `open_question` (0-N): points left open by the decision
- `next_action` (0-N): actions derived from the decision
- Raw: capture `alternatives_considered` in `raw.manual_notes` for later rendering

## Synthesis (LLM pass 2)

MADR-style. Tone rules:
- **Present indicative** ("we adopt X"), not future ("we will adopt")
- Formal technical, no hype
- **Honest** alternatives — only those that were really considered; "none" is valid if there weren't any
- Consequences: explicit positive + negative

### Section gates

- `Context` requires evidence of the problem, constraint, or tradeoff that made the decision necessary.
- `Decision` requires exactly one accepted/proposed decision item.
- `Alternatives considered` must use real alternatives from evidence; if none exist, say so directly.
- `Consequences` requires at least one grounded implication; include negative/open cost when present.
- `Next steps` requires concrete follow-up actions; otherwise omit.
- `Links` requires actual referenced artifacts; otherwise omit.

## Output template

See [examples/adr.good.md](../examples/adr.good.md) for the canonical shape.

Skeleton:
```markdown
# ADR — <short decision title>

**Status:** <proposed | accepted | deprecated | superseded>
**Date:** <YYYY-MM-DD>

## Context
<The problem / situation that required a decision. What was the state of the system? Which constraints mattered?>

## Decision
<The decision taken, in direct and specific prose.>

## Alternatives considered
- **<Alt A>** — <brief description> — <why not chosen>
- **<Alt B>** — <brief description> — <why not chosen>

(or "No substantive alternative was considered" if that's the case)

## Consequences
- **Positive:** <what we gain>
- **Negative:** <what becomes more expensive/complicated>
- **Neutral / open:** <neutral or open implications>

## Next steps
- <derived action 1>
- <derived action 2>

## Links
- <relevant links: reviews, changes, docs, tasks>

---
Sources
- used: ...
- missing: ...
```

## Degradation

| Situation | Behavior |
|---|---|
| No git | Purely decisional ADR (just conversation + manual links) — valid |
| No substantive decision evidence | "Describe the decision in 2-3 lines so I can expand" |
| Multiple decisions detected | Ask which (don't generate a combined ADR) |
| No clear decision | Don't generate — ask for description |

## Do / Don't

**Do:**
- Always make status explicit (never omit)
- Real alternatives ("none" is valid if it was a unanimous/obvious decision)
- Honest consequences — real negatives, not cosmetic ones
- Links to PRs/tasks/docs when relevant
- 1 ADR = 1 main decision (sub-decisions become `open_question` or linked ADRs)

**Don't:**
- Never invent alternatives that weren't considered
- Don't use future ("we will"); use present ("we adopt")
- Don't emit without a clear decision — ask
- Don't do a living tracker's job (if the destination is an evolving tracker task, that's not an ADR)
- Don't mix with handoff (ADR is about **why**, handoff is about **how to continue**)

## Validation checklist

- Is there exactly one main decision?
- Are context, decision, alternatives, consequences, and next steps all grounded in evidence?
- Are alternatives honest, with "none" used if appropriate?
- Does the output match the ADR example shape and stay in one language?
