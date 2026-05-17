# Scribe

> Daily, weekly, ADR, and handoff look like different tasks. They are the same operation with different presets.

```
evidence → semantic normalization → preset → artifact
```

The input doesn't change. The preset does.

## What this is

Scribe is a **preset engine** that turns work evidence into operational artifacts. Each artifact a working person regularly needs — a daily recap, a weekly recap, an ADR, a session handoff, a client update, a retro, a loose-ends triage — is the same pipeline with a different lens.

It is not a summarizer. A summarizer answers "what happened?". Scribe answers what the preset asks.

```
Commodity tool:   git log → summary
Scribe:           evidence → operational briefing → preset → purposeful artifact
```

## Presets

| Preset          | Question it answers                                            | Lens                  |
|-----------------|----------------------------------------------------------------|-----------------------|
| `daily`         | What do I need to communicate today?                           | short temporal        |
| `weekly`        | How do I tell the week's story?                                | long temporal         |
| `adr`           | What decision needs to become memory?                          | decisional            |
| `handoff`       | How do I continue without rebuilding context?                  | continuity            |
| `client-update` | How do I explain progress to someone outside execution?        | external-facing       |
| `retro`         | What did we learn, and what changes next time?                 | retrospective         |
| `loose-ends`    | What is stalled, drifting, or parked — and what decision now?  | triage / hygiene      |

## The proof

The fastest way to understand the thesis is the **from-same-evidence trio**:

- [`examples/demo-evidence.md`](examples/demo-evidence.md) — one bundle of evidence from a design session
- [`examples/from-same-evidence.weekly.md`](examples/from-same-evidence.weekly.md) — that evidence rendered as a weekly recap
- [`examples/from-same-evidence.adr.md`](examples/from-same-evidence.adr.md) — same evidence, ADR lens
- [`examples/from-same-evidence.handoff.md`](examples/from-same-evidence.handoff.md) — same evidence, handoff lens

Three different artifacts. Identical input. Only the preset changes.

Each preset also has a standalone canonical example under `examples/<preset>.good.md` — the renderer mirrors its shape.

## Install

Scribe is a [Claude Code skill](https://docs.claude.com/en/docs/agents/skills). Clone into your skills directory:

```bash
git clone https://github.com/Elesiann/scribe.git ~/.claude/skills/scribe
```

Restart Claude Code. The skill auto-activates on trigger phrases.

## Use

Invoke explicitly:

```
/scribe weekly
/scribe handoff
/scribe loose-ends since last sprint
/scribe adr
```

Or by intent — Scribe maps natural language to a preset:

```
"what did I do today"               → daily
"record this decision"              → adr
"hand off context to next session"  → handoff
"client update"                     → client-update
"what's stalled"                    → loose-ends
```

If the intent is ambiguous, Scribe asks one short question before proceeding.

## Architecture

Scribe is a small engine with a thin interface adapter on top.

```
Interface Adapter (skill — the MVP)
              │
              ▼
PresetInvocation contract
              │
              ▼
Capability detection
              │
              ▼
Evidence gathering (sources gated by capabilities)
              │
              ▼
ContextBundle (neutral contract)
              │
              ▼
Preset renderer → Markdown
              │
              ▼
Source ledger (mandatory)
```

The skill is one adapter. A CLI, a cron job, or an API would be other adapters over the same engine. Internal contracts:

- [`protocols/capability-detection.md`](protocols/capability-detection.md) — how Scribe discovers which sources are available at runtime
- [`protocols/context-bundle.md`](protocols/context-bundle.md) — the neutral contract between evidence and rendering
- [`protocols/source-ledger.md`](protocols/source-ledger.md) — the provenance block every artifact ends with

## Why a source ledger

Every artifact ends with something like:

```
Sources
- used: conversation, git_local
- missing: notion
- denied: atlassian
- error: gh
```

This is what separates Scribe from "ChatGPT summarized this". The ledger is emitted deterministically from probe results — no LLM reasoning, just lookup. **Hallucination-proof by design.**

## Extending

A preset is a markdown file under `presets/` declaring:

1. The question it answers and its lens
2. Required and optional inputs
3. Which `items[]` types it consumes from the `ContextBundle`
4. A skeleton template (by example, not by abstract spec)
5. A validation checklist run before delivery

The matching `examples/<preset>.good.md` is the canonical shape the renderer mirrors. Start from any existing preset — they all follow the same skeleton.

## Non-goals (v1)

- No own OAuth — relies on existing Claude connectors
- No auto-publish — user always confirms destination
- No CLI binary yet — that's a future adapter
- No multi-audience in a single invocation
- No template evolution / learning loop

## License

MIT.
