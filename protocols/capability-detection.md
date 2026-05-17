# Scribe — Capability Detection Protocol

Capability detection is runtime plumbing — it determines, at invocation time, which sources are accessible. **Feed for the engine, not product pitch.**

## Capability taxonomy

Maps runtime capabilities independently of surface branding.

| Capability | Detection method |
|---|---|
| `conversation` | Always available in a conversational surface |
| `filesystem` | Tools `Read` / `Glob` / `Edit` exist |
| `bash` | Tool `Bash` exists |
| `git` | `git rev-parse --is-inside-work-tree` succeeds (requires bash) |
| `gh` | `gh auth status 2>&1 \| grep -q 'Logged in'` (requires bash) |
| `connectors` | At least one `mcp__*` tool exposed |
| `mcp` | MCP tools available |

### Connector-level (subset of `connectors`)

These are common examples, not required sources for the engine:

Each connector detected individually:

- `notion` ← `mcp__notion__notion-search` exists
- `atlassian` ← `mcp__claude_ai_Atlassian__*` exists
- `google_drive` ← `mcp__claude_ai_Google_Drive__*` exists
- `gmail` ← `mcp__claude_ai_Gmail__*` exists
- `calendar` ← `mcp__claude_ai_Google_Calendar__*` exists
- `github_connector` ← `mcp__claude_ai_GitHub__*` exists
- `<custom>` ← other MCPs (e.g., `mcp__<custom>__*`)

## Probe states (fixed 4)

Each source/capability resolves to one of 4 final states:

| State | Meaning |
|---|---|
| `used` | Source available AND probed AND returned useful data |
| `missing` | Source not exposed in this surface (connector not linked, tool not available) |
| `denied` | Source exposed but probe failed due to permission/auth |
| `error` | Source exposed, auth OK, but probe returned an error (500, timeout, malformed) |

**Final states** (those that enter the source ledger) are always one of these 4. Transient internal states (e.g., "probing") may exist during execution but must collapse into one of the 4 by the end.

## Probe rules

### Cheap probes, structured returns

Every capability must **pass a cheap probe** before being considered `used`. Existence alone isn't enough — the probe must return useful data.

### Probe examples

**Git:**
```bash
git rev-parse --is-inside-work-tree 2>/dev/null
# success → used (local git available)
# failure → missing (current workspace is not a local git repo)
```

**Code-hosting CLI example (`gh`):**
```bash
gh auth status 2>&1 | grep -q 'Logged in'
# success → used
# missing binary → missing
# binary present but token expired → denied
```

**Connector/MCP source:** list/fetch one minimal known item.

Examples per connector:

| Connector | Probe | Success → | Failure → |
|---|---|---|---|
| notion | `mcp__notion__notion-search` with trivial query, page_size=1 | `used` | depending on error: `denied` / `error` / `missing` |
| atlassian | `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources` | `used` | analogous |
| google_drive | list 1 file | `used` | analogous |

### Classification logic

When a probe fails, classify by error type:

- Tool does not exist in schema → `missing`
- Tool exists, returns 401/403 → `denied`
- Tool exists, returns 5xx/timeout/malformed → `error`
- Tool exists, returns success with empty data → subjective; default to `used` with item count 0

## Conversation is always `used`

In any conversational surface, `conversation` is `used` by default. Never `missing`.

## Reporting to user

**Critical distinction:** there are **two capability states** reporting must separate:

1. **Exposure** (pre-probe): does the tool/connector exist in the surface? E.g., does `mcp__notion__*` appear in the schema?
2. **Probe result** (post-attempt): did a cheap probe execute and return useful data?

Only the **second** authorizes reporting a source as `used`. Exposure alone → `exposed` or `available in surface`.

### Recommended format

```
Surface exposes:
  ✅ conversation, bash, filesystem
  ✅ git (tool), gh (CLI authenticated)
  🔌 notion, atlassian (connectors exposed — not probed yet)

Probed for this preset:
  ✅ git_local (workspace: /path/to/project, 31 changes in period) — used
  ✅ gh (13 review items merged in period) — used
  ⚪ connected trackers (skipped — code evidence sufficient for the preset)

Preset: weekly (explicit) — proceeding
```

**Separated legends:**

Exposure-level:
- ✅ tool/connector exposed, ready to probe if needed
- ⚠ not exposed in this surface

Probe-level (only one that feeds the ledger):
- ✅ **used** — probed AND returned useful data
- ⚠ **missing** — probed, no data returned (e.g., 0 commits in period)
- 🚫 **denied** — probed, failed due to auth/permission
- ❌ **error** — probed, failed due to technical error (500, timeout)
- ⚪ **skipped** — chose not to probe (does NOT enter the ledger, only appears in reporting)

## Source Access Discipline

**Before collecting evidence from a requested source, Scribe must classify whether that source is actually accessible in the current surface.**

If the user requests a source that is not accessible:

- state the limitation clearly
- do not pretend the source was used
- do not silently switch to a weaker source as if equivalent
- suggest available alternatives
- ask whether to continue, unless graceful fallback was already requested

**Rule:** avoid speculative or brittle fallback paths unless the user explicitly approves them.

### Explicit source requirement rule

If the user explicitly names a source, that source becomes a requirement for the current invocation until the user relaxes it.

Operational behavior:

1. detect the requested source from the latest user instruction
2. probe that source first
3. if available, proceed normally
4. if unavailable, ask for fallback approval before gathering from other sources
5. only after approval may Scribe substitute weaker or different evidence sources

Examples of explicit source requirements:

- `"based on my commits in github"` → code-hosting evidence (`gh` or `github_connector`)
- `"use local git"` → `git_local`
- `"use notion pages from this week"` → a connected workspace/tracker source (`notion`)
- `"just use this conversation"` → `conversation`

This rule matters because tracker/docs evidence is not equivalent to code-hosting history, and conversation is not equivalent to completed work.

### Example

User asks for code-hosting history, but no connector, CLI, or local git history is available.

**Good:**
> "I don't currently have access to the requested code-hosting evidence in this surface. I can continue with connected trackers + conversation history, or you can run this where local code evidence is available."

**Bad:**
> "I'll try a random public API call and infer from partial data."

Bad is bad because: it blurs provenance, produces hallucinated-feeling output, and silently weakens the artifact below the user's expectation.

## Source Substitution Rule

**A fallback source is not equivalent to the requested source unless it provides the same kind of evidence.**

Examples:

- A code-hosting connector, CLI, and local git can all provide change/review evidence, but with different coverage
- A tracker/docs connector can provide work notes, but it is not a substitute for change history
- Conversation can provide intent and context, but it is not proof of merged work

**When substituting sources, Scribe must label the substitution** in the source ledger or in a short note before rendering.

Example label in the final ledger:

```
Sources
- used: conversation (substituting for git — not available in this surface)
- missing: git_local, gh
```

Or in a pre-render note:

> "Git not available in this surface; rendering `weekly` from conversation context only. Artifact confidence is lower than a git-backed run."

When the source was explicitly requested, add one more rule:

- **substitution requires user approval first**

## Graceful degradation

Scribe never fails because a source is absent. It degrades:

1. **Never fail because a source is missing.** If the preset requires specific evidence that is absent, degrade to conversation-only or ask for manual context.
2. **Always flag in the ledger** — missing/denied/error become visible.
3. **Ask the user before operating conversation-only.** Before running with just `conversation`, confirm: "Only conversation is available. Paste additional context or should I proceed with what I have?"
4. **Document the preset's minimum signal expectations** in the preset file. The ledger shows what was missing versus what was expected.

If a source was explicitly requested, graceful degradation only starts **after** the user approves fallback.

## Surface notes (non-prescriptive)

Capability detection is runtime-driven — it does not assume product branding. Common patterns observed:

- **Claude Code (CLI/IDE):** typically has full `conversation + bash + filesystem + connectors`
- **Claude Desktop / claude.ai web:** typically has `conversation + connectors`, without `bash` / `filesystem`
- **Future surfaces:** mapped by the same protocol, no special-casing

The engine doesn't change. What changes is the set of sources marked `used` per invocation.
