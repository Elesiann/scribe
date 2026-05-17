# Preset: weekly

**Lens:** long temporal. Consolidates a week (or custom period) of work across multiple sources.

**Default audience:** tech lead / self for 1:1 prep / sprint review.

## Contract

| Aspect | Value |
|---|---|
| Input | User asks for a weekly recap (explicit or paraphrased) |
| Output | Multi-section Markdown with highlights, themes, decisions, blockers, where we stopped, next steps |
| Scope | Temporal window (default 7 days) |
| Destination | Inline (default) — can save to file or suggest a workspace destination (never auto-publishes) |

## Required inputs

Ask only if not clear from the prompt:

- **period** — time window
  - Default: last 7 days (`git log --since "1 week ago"`)
  - Accepts: `"since monday"`, `"last 14 days"`, `"current sprint"`, `"2026-02-15..2026-02-20"`

## Optional inputs

- **destination** — `inline` (default) | `file` | `workspace-destination-suggestion`
- **resource_scope** — override for the current code workspace or connected work context (useful in multi-workspace setups)

## Sources (priority order)

| Source | Role | Required in mode |
|---|---|---|
| local code workspace | primary change evidence when available | rich, minimal |
| work-tracking connector | organizational / task evidence | rich |
| issue tracker connector | issue / planning evidence | rich |
| code-hosting review signal | merge / review evidence | rich |
| conversation | decisional signals from current session | always |

## Minimum viable signal

For `weekly` to render quality, at least **one** of the following must hold:
- `git_local` `used` + >= 3 meaningful changes in the period
- a connected tracker/issue source `used` + >= 2 items in the period
- `conversation` `used` + substantive context (pasted notes or decisional discussion)

Below this, Scribe asks: "Not enough signal for a weekly. Paste extra notes or would you rather run `daily` with what we have?"

## Primary item types (ContextBundle)

Weekly renderer primarily consumes:
`delivery | decision | risk | blocker | next_action | work_item`

Secondary fields: `summary.themes`, `summary.where_we_stopped`.

## Gathering (capability-gated)

### git (primary, if capability `git` is present)

**Authorship: assume multiple possible emails per user.** Using global `user.email` alone is fragile — a user may have different identities per workspace. Strategy:

1. **Detect candidate emails:** `git config --get-all user.email` + `git config --global --get user.email` (union, dedup)
2. **Optional:** accept override via `--author-email` or ask the user in the clarify step when >1 email is detected
3. **Robust fallback:** OR filter via multiple `--author` flags: `git log --author="email1" --author="email2" ...`

```bash
PERIOD="<period_start>"  # e.g., "2026-02-15" or "1 week ago"

# Detect candidate emails (per-repo + global)
EMAILS=$(cd "$REPO" && { git config --get-all user.email; git config --global --get user.email; } | sort -u)

# Multi-author filter
AUTHOR_FLAGS=""
for email in $EMAILS; do
  AUTHOR_FLAGS="$AUTHOR_FLAGS --author=$email"
done

# commits by author in period
git log --since "$PERIOD" $AUTHOR_FLAGS \
  --format='%h|%ai|%s|%an|%ae' --date=iso | head -50

# diff stat over the period
FIRST_SHA=$(git log --since "$PERIOD" $AUTHOR_FLAGS --format='%h' --reverse | head -1)
[ -n "$FIRST_SHA" ] && git diff --stat "${FIRST_SHA}^..HEAD" 2>/dev/null

# current branch + status
git rev-parse --abbrev-ref HEAD
git status --short
```

**Parse into:** `{ sha, date, subject, author, files_touched_count }`.
**Cap:** 50 commits.

### Multi-workspace scan

If the user asks for a cross-workspace recap ("all the codebases/workspaces I touched this week"), scanning requires discovery. Two strategies:

**(Preferred) Pre-configured list:** skill maintains a local list of code workspaces (1 path per line). User fills it once; scan becomes fast and deterministic.

**(Fallback) Filesystem scan:** `find <roots> -maxdepth 6 -type d -name .git` with bounded `<roots>` (e.g., `~/projects`, `~/work`, `~/src`). **Avoid scanning the entire home directory** — expensive and non-deterministic.

v1.5 will consider candidate caching with TTL.

### Connected work-tracking connector (example: Notion)

Use `mcp__notion__notion-search`:
- `query`: empty string (returns most recent)
- `filters.created_date_range.start_date`: period_start
- `page_size`: 20

Follow up with `mcp__notion__notion-fetch` on the top 10 results to extract status + completed_at.

**Parse into:** `{ id, title, status, completed_at, excerpt }`.
**Cap:** 20 items.

### Connected issue tracker (example: Atlassian)

Use `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`:
```
assignee = currentUser()
  AND resolved >= "<period_start>"
  AND resolution != Unresolved
ORDER BY resolved DESC
```
**Cap:** 20 items.

### Code-hosting review signal (example: `gh` CLI)

⚠ **Known bug:** `--json repository` **does not exist** on `gh pr list`. Valid fields include `number, title, mergedAt, url, author, headRepository, state`. To identify each review item's workspace, use `url` (parse the path) OR run `gh pr list` per local workspace (`cd <repo> && gh pr list`).

**Option A — global search (parse workspace from URL):**
```bash
gh pr list --author @me --state merged \
  --search "merged:>=<period_start>" \
  --json number,title,mergedAt,url,headRepository --limit 20
```

**Option B — per-workspace (more reliable in multi-workspace):**
```bash
for repo in <repo_list>; do
  cd "$repo"
  gh pr list --author @me --state merged \
    --search "merged:>=<period_start>" \
    --json number,title,mergedAt,url --limit 10
  cd -
done
```

**Cap:** 20 items total (not per repo).

### conversation (always)

Filter the active conversation for:
- Decision mentions ("we chose X", "going with Y")
- Blockers ("stuck on", "waiting on", "review not yet approved")
- Intent markers ("next step", "tomorrow", "TODO")

Emit a bullet list of relevant user statements. Skip trivia.

## Extraction (LLM pass 1)

Produce in `extracted` / `items[]` of the ContextBundle:

### themes
Cluster changes + tasks + review items into 2–5 thematic groups. Labels come from titles, linked work, and visible intent.

Format:
```
- Sync engine hardening (9 changes, 3 tasks, 1 review item) — 47% of period
- Import path closeout (3 changes, 1 task) — 18%
- Customer-specific integration work (4 changes) — 23%
- Infra / misc (1 change, 1 task) — 12%
```

**Rule:** percentages should add up to roughly 100%; use item count as weight.

### decisions_mentioned
Technical decisions made in the period. Extract from:
- change messages with explicit `Why:`
- connected docs/tracker pages that capture decisions
- linked docs or wiki references
- Conversation (if user mentioned a choice)

Format: `- <decision in 1 line> — source: <change-id | task-id | conversation>`

### blockers
Stalled items, failures, expirations. Examples:
- `In Progress` tasks > 3 days without execution signal
- CI failures mentioned
- Pending review approvals / unmet dependencies
- Unreviewed open review items

Format: `- <short description> — source: <ref>`

### next_actions_implied
TODOs, next steps, deferred tasks. Extract from:
- changes with "TODO" / "next:"
- open (not merged) review items
- connected tracker items in "To Do" created in the period
- Conversation ("tomorrow I...", "next week...")

## Synthesis prompt (LLM pass 2)

After the bundle + extracted are ready, synthesize following the output template. Rules:

- Use `themes` as Level-2 headers
- **Don't list raw commits/tasks** — synthesize into thematic narrative
- Cite specifics: change IDs, review IDs, task IDs, or links **inline** when appropriate
- Surface blockers explicitly, don't bury them
- "Where we stopped" reflects live state (current branch, open review, in-progress task, or equivalent)
- If unresolved WIP deserves investigation, mention it briefly and suggest `loose-ends`; do not triage item-by-item here
- Tone: plain engineering English

## Output template

See [examples/weekly.good.md](../examples/weekly.good.md) for the canonical shape.

Skeleton:
```markdown
# Weekly Recap — <period>

<Context in 1-2 lines>

## Highlights
- <3-5 bullets: outcomes, not counts>

## By theme

### <Theme 1> (<N items, X% of period>)
<narrative synthesis; cite IDs/links inline>

### <Theme 2> (...)
...

## Decisions
- <decision> (<source>)

## Blockers / risks
- <what's stalled and why>

## Where we stopped
<live state — current branch, open review, in-progress task, or equivalent>

## Next steps
- <what makes sense for next week>

---
<source ledger — format in protocols/source-ledger.md>
```

## Degradation

| Situation | Behavior |
|---|---|
| current workspace has no local code history | Ask: "Point to another workspace or proceed with connected sources only?" |
| Local code history OK, no connected trackers available | Proceeds in `minimal` mode, flagged in ledger |
| No technical sources | `degraded` mode: ask "paste this week's notes"; emit ⚠ note |

## Do / Don't

**Do:**
- Normalize dates to local timezone for display
- Cluster aggressively — 20 commits should not yield 20 bullets
- Omit sections without evidence (no "no data available")
- Surface unresolved WIP briefly when it matters
- Suggest `loose-ends` when the user clearly needs investigation instead of recap

**Don't:**
- Don't list raw commits in the output
- Don't put raw IDs in headers — inline citations only
- Don't comment on periods of inactivity
- Don't mix with another preset's template even if the user mentions both — ask which
- Don't do evidence-heavy triage; that belongs to `loose-ends`

## Validation checklist

- Does the artifact tell the story of the period rather than list events?
- Are themes clustered, with blockers and next steps explicit?
- Is unresolved WIP only mentioned briefly, without turning into triage?
- Does the output match the weekly example shape and stay in one language?
