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
  - Default: last 7 days
  - Accepts: `"since monday"`, `"last 14 days"`, `"current sprint"`, `"2026-02-15..2026-02-20"`

## Optional inputs

- **destination** — `inline` (default) | `file` | `workspace-destination-suggestion`
- **resource_scope** — override for the current code workspace or connected work context (useful in multi-workspace setups)

## Source selection

Weekly is harness-agnostic. Source selection is driven by the user's scope and
the available evidence, not by a fixed preference for code history.

| Source family | Role |
|---|---|
| work-tracking / issue systems | planned, active, completed, blocked, or deferred work |
| session traces / conversation | session-level decisions, attempts, corrections, blockers, and stated outcomes |
| local code workspace | concrete implementation movement when code work is in scope |
| code-hosting review signal | PR/review/merge state when review flow is the relevant work surface |
| docs / filesystem / connected workspaces | specs, notes, artifacts, or other deliverables that explain what changed |

Rules:

- Use the sources that best prove what was done for the requested scope.
- Do not treat git/GitHub as privileged unless the user asks for code-hosting
  evidence or the work is primarily code delivery.
- Do not treat conversation as delivery proof by itself unless the user provides
  substantive notes or the work happened inside the active session.
- If multiple sources describe the same work, merge them into one item and keep
  the strongest evidence refs.

## Minimum viable signal

For `weekly` to render quality, at least **one** of the following must hold:
- a work-tracking / issue / code-hosting source shows completed or materially advanced work in the period
- `git_local` shows meaningful implementation movement for a code-scoped weekly
- `session_trace` or `conversation` contains substantive work-session evidence: outcomes, attempts, blockers, decisions, and current state
- manual notes or docs describe concrete work done, not only intentions

Below this, Scribe asks: "Not enough signal for a weekly. Paste extra notes or would you rather run `daily` with what we have?"

## Primary item types (ContextBundle)

Weekly renderer primarily consumes:
`delivery | decision | risk | blocker | next_action | work_item`

Secondary fields: `summary.themes`, `summary.where_we_stopped`.

## Gathering (capability-gated)

### session_trace (if available and relevant)

Follow [../protocols/session-trace.md](../protocols/session-trace.md). Use it
when the week's work happened inside agent sessions or when session state is the
best evidence of outcomes, attempts, blockers, and where work stopped.

### git (if available and relevant)

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

If the user asks for a cross-workspace code recap ("all the codebases/workspaces I touched this week"), scanning requires discovery. Two strategies:

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

### Code-hosting contribution signal (example: `gh` CLI)

⚠ **Known bug:** `--json repository` **does not exist** on `gh pr list`. Valid fields include `number, title, mergedAt, url, author, headRepository, state`. To identify each review item's workspace, use `url` (parse the path) OR run `gh pr list` per local workspace (`cd <repo> && gh pr list`).

Collect code-hosting evidence only when code review / PR flow is relevant to the
requested weekly. Do not limit the signal to PRs authored by the current user:
reviews, assigned PRs, commented PRs, and open PRs may be the more important
work for the period.

**Option A — authored/merged work (parse workspace from URL):**
```bash
gh pr list --author @me --state merged \
  --search "merged:>=<period_start>" \
  --json number,title,mergedAt,url,headRepository --limit 20
```

**Option B — involved/review work (per workspace, broader contribution signal):**
```bash
gh pr list --state all \
  --search "involves:@me updated:>=<period_start>" \
  --json number,title,state,mergedAt,updatedAt,url,author,reviewDecision --limit 20
```

**Option C — per-workspace authored work (more reliable in multi-workspace):**
```bash
for repo in <repo_list>; do
  cd "$repo"
  gh pr list --author @me --state merged \
    --search "merged:>=<period_start>" \
    --json number,title,mergedAt,url --limit 10
  cd -
done
```

**Cap:** 20 items total per query family, then deduplicate by URL. Prefer the
items that best explain work done in the period over raw volume.

### conversation (always)

Filter the active conversation for:
- Outcomes or completed work ("shipped X", "finished Y", "implemented Z")
- Attempts that changed the work path ("tried X, failed because Y")
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
| requested source is unavailable | Ask before fallback, following explicit source requirement rules |
| current workspace has no local code history, but connected/session/manual evidence is strong | Proceed from those sources; do not treat missing git as degraded by itself |
| only weak intent-level conversation exists | Ask for notes or suggest `daily` if the scope is too thin |
| no source contains evidence of work done | Do not generate a weekly; ask for a source or notes |

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
