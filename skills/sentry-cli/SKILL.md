---
name: sentry-cli
version: 0.34.0
description: Guide for using the Sentry CLI to interact with Sentry from the command line. Use when the user asks about viewing issues, events, projects, organizations, making API calls, or authenticating with Sentry via CLI.
requires:
  bins: ["sentry"]
  auth: true
disallowed-tools: "Bash(sentry project delete:*),Bash(sentry trial start:*)"
---

# Sentry CLI Usage Guide

Help users interact with Sentry from the command line using the `sentry` CLI.

## Agent Guidance

Best practices and operational guidance for AI coding agents using the Sentry CLI.

### Key Principles

- **Just run the command** — the CLI handles authentication and org/project detection automatically. Don't pre-authenticate or look up org/project before running commands. If auth is needed, the CLI prompts interactively.
- **Prefer CLI commands over raw API calls** — the CLI has dedicated commands for most tasks. Reach for `sentry issue view`, `sentry issue list`, `sentry trace view`, etc. before constructing API calls manually or fetching external documentation.
- **Use `sentry schema` to explore the API** — if you need to discover API endpoints, run `sentry schema` to browse interactively or `sentry schema <resource>` to search. This is faster than fetching OpenAPI specs externally.
- **Use `sentry issue view <id>` to investigate issues** — when asked about a specific issue (e.g., `CLI-G5`, `PROJECT-123`), use `sentry issue view` directly.
- **Use `--json` for machine-readable output** — pipe through `jq` for filtering. Human-readable output includes formatting that is hard to parse.
- **The CLI auto-detects org/project** — most commands work without explicit targets by checking `.sentryclirc` config files, scanning for DSNs in `.env` files and source code, and matching directory names. Only specify `<org>/<project>` when the CLI reports it can't detect the target or detects the wrong one.
- **Read the CLI's error message before retrying** — when a command fails, the error usually names the correct next invocation (longer `--period`, the org-wide form, the missing positional, the right dataset). Don't guess; the error often is the answer. Exit codes are semantic (see the table below).
- **Trust the `[in-app]` markers in stack traces** — `sentry issue view` annotates app frames with `[in-app]`. Use that flag to separate signal from vendor noise rather than maintaining your own regex of `/vendor/`-style paths.
- **Be sceptical of Sentry's repo attribution** — code mappings can be auto-generated and wrong (multiple repos mapped to one project; the wrong one "winning"). Seer's `issue explain` reaches *beyond* code mappings via content search and will sometimes name a repo that isn't even in the project's mapping list. Before quoting file paths or pulling a specific local repo, verify with: `sentry api /api/0/projects/<org>/<project>/seer/preferences/ --json | jq '.code_mapping_repos[].name'`. If multiple repos appear, *ask the user* which is correct — don't guess by name similarity.
- **Reconcile against local code loudly** — when reporting findings from `issue view`, state the local checkout's path, branch, last-commit age and clean/dirty status *before* quoting any line numbers. Stale clones (months-old branches) are common; even when paths match, deployed code may have moved on. See [references/workflows/investigate-an-issue.md](references/workflows/investigate-an-issue.md).
- **Volume math reframes scary numbers** — divide event count by days-since-first-seen. A "5648 events" issue first seen 18 months ago is ≈10/day: a small stable trickle, not a fire. Note the rate in any verdict — it changes urgency framing.

### First action: check the token's scopes (once per session)

Before doing real investigation work, inspect what the active token can actually do:

```bash
sentry api /api/0/ --json | jq -r '.auth.scopes[]'
```

If `jq` isn't installed, the `--fields` flag returns a nested JSON blob you can read directly (no piping needed):

```bash
sentry api /api/0/ --json --fields auth.scopes      # jq-free alternative
```

Compare against the **recommended minimum** for this skill (full table and rationale in the Authentication section further down):

```
org:read   project:read   event:read   event:write   team:read   member:read
```

If the token has **any other write/admin scopes** — particularly `project:admin`, `project:write`, `event:admin`, `org:write`, `org:admin`, `team:write`, `team:admin`, `member:write`, `member:admin` — surface this to the user **before doing real work**, in roughly this form:

> ⚠️ Your Sentry token has broader scopes than this skill needs:
> - `project:admin` — would allow a Sentry project to be deleted via API
> - `project:write` — broad project mutations
> - `team:write` — team mutations
>
> I'll stay within read + issue-status-mutation territory regardless, but defence-in-depth is stronger with a least-privileged token (see the Authentication section). Say "carry on" if you're happy proceeding with the current token, or rotate first.

Then **wait for acknowledgement** before continuing. Don't re-run the scope check before every command — once per session is enough. If the user explicitly rotates their token mid-session, re-check.

This check is the agent's responsibility because the consequences of a runaway model (especially smaller/cheaper models picked for cost) with an admin-scoped token are serious and irreversible. Naming the specific extra scopes lets the user make an informed call.

### Design Principles

The `sentry` CLI follows conventions from well-known tools — if you're familiar with them, that knowledge transfers directly:

- **`gh` (GitHub CLI) conventions**: The `sentry` CLI uses the same `<noun> <verb>` command pattern (e.g., `sentry issue list`, `sentry org view`). Flags follow `gh` conventions: `--json` for machine-readable output, `--fields` to select specific fields, `-w`/`--web` to open in browser, `-q`/`--query` for filtering, `-n`/`--limit` for result count.
- **`sentry api` mimics `curl`**: The `sentry api` command provides direct API access with a `curl`-like interface — `--method` for HTTP method, `--data` for request body, `--header` for custom headers. It handles authentication automatically. If you know how to call a REST API with `curl`, the same patterns apply.

### Context Window Tips

- Use `--json --fields` to select specific fields and reduce output size. Run `<command> --help` to see available fields. Example: `sentry issue list --json --fields shortId,title,priority,level,status`
- Use `--json` when piping output between commands or processing programmatically
- Use `--limit` to cap the number of results (default is usually 10–100)
- Prefer `sentry issue view PROJECT-123` over listing and filtering manually
- Use `sentry api` for endpoints not covered by dedicated commands
- **`sentry issue view` output can be large** (50–100KB for issues with long stack traces) and the Read/Bash tooling will typically spill it to a tool-results file with a preview. When this happens, read the saved file in slices (use `offset`/`limit` on the Read tool) rather than re-running the command. The `[in-app]` markers tell you which frames to focus on — vendor frame blocks can be skipped past quickly.

### Safety Rules

- **`sentry project delete` and `sentry trial start` are blocked at the frontmatter level** (`disallowed-tools`) — they cannot run from this skill at all. If a user genuinely needs either, they should run it directly in their terminal, not via Claude. Don't try to work around this with `sentry api`; do not propose either action as a step.
- For other mutations (e.g. `sentry issue resolve`, `sentry release delete`, `sentry dashboard delete`, `sentry api ... --method DELETE/PUT/POST`), confirm with the user before running. Verify the org/project context in the command output before any further changes.
- Never store or log authentication tokens — the CLI manages credentials automatically.
- If the CLI reports the wrong org/project, override with explicit `<org>/<project>` arguments.
- **Git is strictly read-only during investigation.** Only run inspection commands: `git log`, `git show`, `git status`, `git diff`, `git branch`, `git remote -v`, `git rev-parse`, `git blame`. **Never run** `git fetch`, `git pull`, `git checkout`, `git stash`, `git reset`, `git merge`, `git rebase`, or anything else that mutates refs or working-tree state. The developer is likely mid-flow on another task and the last thing they need is sudden remote output or an unexpected working-tree change. If a ref doesn't exist locally (e.g. `origin/main` was never fetched), work without it — don't run `git fetch` to "freshen up".

### Exit Codes

The CLI uses semantic exit codes. Key ranges for agents:

| Range | Meaning | Agent Action |
|-------|---------|-------------|
| 0 | Success | Proceed normally |
| 10–19 | Auth error | Prompt user to run `sentry auth login` |
| 20–29 | Input error | Check command arguments and retry |
| 30–39 | API error | Retry or report to user |
| 40–49 | Feature unavailable | Inform user about plan/settings |
| 50–59 | Operation error | Report to user |
| 60–69 | Command-specific | Check stderr for details |

See [Exit Codes](/exit-codes/) for the complete reference.

### Workflow Patterns

For richer scenarios with decision branches and reconciliation steps, see the playbooks in `references/workflows/`:

- **[Investigate an issue end-to-end](references/workflows/investigate-an-issue.md)** — the "look into PROJX-1234" flow, with local-checkout reconciliation, Seer comparison, volume math, and a worked example. Use this when the user pastes an issue ID or link.

The short bash snippets below are quick command-sequences for the common shapes. Reach for the playbook above for any non-trivial investigation.

#### Investigate an Issue (quick sequence)

```bash
# 1. Find the issue (auto-detects org/project from DSN or config)
sentry issue list --query "is:unresolved" --limit 5

# 2. Get details — output includes [in-app] markers on application frames
sentry issue view PROJECT-123

# 3. Verify which repo Sentry thinks this is (code mappings can be wrong)
sentry api /api/0/projects/<org>/<project>/seer/preferences/ --json \
  | jq '.code_mapping_repos[].name'

# 4. Get AI root cause analysis (10-60s)
sentry issue explain PROJECT-123

# 5. Get a fix plan (10-60s)
sentry issue plan PROJECT-123
```

#### Explore Traces and Performance

```bash
# 1. List recent traces (auto-detects org/project)
sentry trace list --limit 5

# 2. View a specific trace with span tree
sentry trace view abc123def456...

# 3. View spans for a trace
sentry span list abc123def456...

# 4. View logs associated with a trace
sentry trace logs abc123def456...
```

#### Stream Logs

```bash
# Stream logs in real-time (auto-detects org/project)
sentry log list --follow

# Filter logs by severity
sentry log list --query "severity:error"
```

#### Explore the API Schema

```bash
# Browse all API resource categories
sentry schema

# Search for endpoints related to a resource
sentry schema issues

# Get details about a specific endpoint
sentry schema "GET /api/0/organizations/{organization_id_or_slug}/issues/"
```

#### Manage Releases

```bash
# Create a release — version must match Sentry.init({ release }) exactly
sentry release create my-org/1.0.0 --project my-project

# Associate commits via repository integration (needs local git checkout)
sentry release set-commits my-org/1.0.0 --auto

# Or read commits from local git history (no integration needed)
sentry release set-commits my-org/1.0.0 --local

# Mark the release as finalized
sentry release finalize my-org/1.0.0

# Record a production deploy
sentry release deploy my-org/1.0.0 production
```

**Key details:**
- The positional is `<org-slug>/<version>`. In `sentry release create sentry/1.0.0`, `sentry` is the org and `1.0.0` is the version — the slash separates org from version, it is not part of the version string.
- The **version** must match the `release` value in `Sentry.init()`. If your SDK uses `"1.0.0"`, the command must use `org/1.0.0`.
- `--auto` requires a Sentry repository integration (GitHub/GitLab/Bitbucket) **and** a local git checkout. It matches your `origin` remote against Sentry's repo list. Without a checkout, use `--local`.
- With no flag, `set-commits` tries `--auto` first and falls back to `--local` on failure.

#### Arbitrary API Access

```bash
# GET request (default)
sentry api /api/0/organizations/my-org/

# POST request with data
sentry api /api/0/organizations/my-org/projects/ --method POST --data '{"name":"new-project","platform":"python"}'
```

### Dashboard Layout

Sentry dashboards use a **6-column grid**. When adding widgets, aim to fill complete rows (widths should sum to 6).

Display types with default sizes:

| Display Type | Width | Height | Category | Notes |
|---|---|---|---|---|
| `big_number` | 2 | 1 | common | Compact KPI — place 3 per row (2+2+2=6) |
| `line` | 3 | 2 | common | Half-width chart — place 2 per row (3+3=6) |
| `area` | 3 | 2 | common | Half-width chart — place 2 per row |
| `bar` | 3 | 2 | common | Half-width chart — place 2 per row |
| `table` | 6 | 2 | common | Full-width — always takes its own row |
| `stacked_area` | 3 | 2 | specialized | Stacked area chart |
| `top_n` | 3 | 2 | specialized | Top N ranked list |
| `categorical_bar` | 3 | 2 | specialized | Categorical bar chart |
| `text` | 3 | 2 | specialized | Static text/markdown widget |
| `details` | 3 | 2 | internal | Detail view |
| `wheel` | 3 | 2 | internal | Pie/wheel chart |
| `rage_and_dead_clicks` | 3 | 2 | internal | Rage/dead click visualization |
| `server_tree` | 3 | 2 | internal | Hierarchical tree display |
| `agents_traces_table` | 3 | 2 | internal | Agents traces table |

Use **common** types for general dashboards. Use **specialized** only when specifically requested. Avoid **internal** types unless the user explicitly asks.

Available datasets: `spans` (default), `tracemetrics`, `discover`, `issue`, `error-events`, `logs`. Run `sentry dashboard widget --help` for dataset descriptions, query formats, and examples.

**Row-filling examples:**

```bash
# 3 KPIs filling one row (2+2+2 = 6)
sentry dashboard widget add <dashboard> "Error Count" --display big_number --query count
sentry dashboard widget add <dashboard> "P95 Duration" --display big_number --query p95:span.duration
sentry dashboard widget add <dashboard> "Throughput" --display big_number --query epm

# 2 charts filling one row (3+3 = 6)
sentry dashboard widget add <dashboard> "Errors Over Time" --display line --query count
sentry dashboard widget add <dashboard> "Latency Over Time" --display line --query p95:span.duration

# Full-width table (6 = 6)
sentry dashboard widget add <dashboard> "Top Endpoints" --display table \
  --query count --query p95:span.duration \
  --group-by transaction --sort -count --limit 10
```

### Quick Reference

#### Time filtering

Use `--period` (alias: `-t`) to filter by time window:

```bash
sentry trace list --period 1h
sentry span list --period 24h
sentry span list -t 7d
```

**Defaults differ per command** — and "no results" often just means "extend the window":

| Command | Default `--period` |
|---|---|
| `issue list` | 90d (only issues with activity in the last 90 days) |
| `explore` | 24h |
| `trace logs` | 14d |
| `trace list` | varies; check `--help` |

When an investigation comes up empty, try a wider window (`--period 30d`, `--period 90d`) before concluding there's no data. The CLI's empty-state messages will usually suggest this — read them.

#### Caching and freshness

Many list/view commands cache results briefly and append a footer like:

```
cached · 3m ago · use -f to refresh
```

Use `-f` / `--fresh` to bypass the cache and force a fresh fetch:

```bash
sentry issue list -f
sentry project list --fresh
```

When investigating an issue that may have just changed state (resolved/reopened/new event), or when verifying the result of a mutation you just made (`sentry issue resolve …`), pass `-f` on the follow-up list/view.

#### Scoping to an org or project

Org and project are positional arguments following `gh` CLI conventions:

```bash
sentry trace list my-org/my-project          # explicit
sentry issue list my-org/my-project          # explicit
sentry span list my-org/my-project/abc123def456...
```

The **trailing slash on `<org>/`** is significant and unlocks org-wide listing:

```bash
sentry issue list my-org/                    # ALL projects in org
sentry repo list my-org/                     # all repos in org
sentry project list my-org/                  # all projects in org
```

Without the trailing slash, a bare slug is treated as a project name search across all orgs (e.g. `sentry issue list myapp` looks for a project called `myapp`). If the bare slug happens to match an org, the CLI will list across that org and emit a warning — better to be explicit with the trailing `/`.

To search across all projects in an org from `issue list`, the trailing-slash form is your friend — far cleaner than running a `sentry api` against `/api/0/organizations/<org>/issues/`.

#### Listing spans in a trace

Pass the trace ID as a positional argument to `span list`:

```bash
sentry span list abc123def456...
sentry span list my-org/my-project/abc123def456...
```

#### Dataset names for the Events API

When querying the Events API (directly or via `sentry api`), valid dataset values are: `spans`, `transactions`, `logs`, `errors`, `discover`.

### Common Mistakes

- **Wrong issue ID format**: Use `PROJECT-123` (short ID), not the numeric ID `123456789`. The short ID includes the project prefix.
- **Pre-authenticating unnecessarily**: Don't run `sentry auth login` before every command. The CLI detects missing/expired auth and prompts automatically. Only run `sentry auth login` if you need to switch accounts.
- **Missing `--json` for piping**: Human-readable output includes formatting. Use `--json` when parsing output programmatically.
- **Specifying org/project when not needed**: Auto-detection resolves org/project from `.sentryclirc` config files, DSNs, env vars, and directory names. Let it work first — only add `<org>/<project>` if the CLI says it can't detect the target or detects the wrong one.
- **Confusing `--query` syntax**: The `--query` flag uses Sentry search syntax (e.g., `is:unresolved`, `assigned:me`), not free text search.
- **Not using `--web`**: View commands support `-w`/`--web` to open the resource in the browser — useful for sharing links.
- **Fetching API schemas instead of using the CLI**: Prefer `sentry schema` to browse the API and `sentry api` to make requests — the CLI handles authentication and endpoint resolution, so there's rarely a need to download OpenAPI specs separately.
- **Release version mismatch**: The `org/version` positional is `<org-slug>/<version>`, where `org/` is the org, not part of the version. `sentry release create sentry/1.0.0` creates version `1.0.0` in org `sentry`. If your `Sentry.init()` uses `release: "1.0.0"`, this is correct. Don't double-prefix like `sentry/myapp/1.0.0`.
- **Running `set-commits --auto` without a git checkout**: `--auto` needs a local git repo to discover the origin remote URL and HEAD commit. In CI, ensure `actions/checkout` with `fetch-depth: 0` runs before `set-commits --auto`.
- **Using `sentry api` when CLI commands suffice**: `sentry issue list --json` already includes `shortId`, `title`, `priority`, `level`, `status`, `permalink`, and other fields at the top level. Some fields like `count`, `userCount`, `firstSeen`, and `lastSeen` may be null depending on the issue. Use `--fields` to select specific fields and `--help` to see all available fields. Only fall back to `sentry api` for data the CLI doesn't expose.
- **Confusing `--metric` with `--field` on `sentry explore`**: `--metric` / `-m` is **only** for `--dataset metrics` (named custom metrics like `llm.token_usage`), and passing it implicitly forces `--dataset metrics` — overriding any other `--dataset` you set. For aggregates of span/event fields like `p95(span.duration)` or `count()`, use `-F "p95(span.duration)"`, not `-m span.duration`. Rule of thumb: if you'd write it as a function call (`p95(...)`, `count_unique(...)`), it goes in `--field`. If it's a bare named metric, it goes in `--metric`.
- **Forgetting the trailing slash on `<org>/`**: `sentry issue list my-org` (no slash) is a project-name search across orgs; `sentry issue list my-org/` (with slash) lists all issues in the org. The two have very different behaviour. See "Scoping to an org or project" above.
- **Not knowing the `--period` defaults**: `issue list` defaults to 90 days, `explore` to 24h, `trace logs` to 14d. "No results" often means "the window is too narrow" — try `--period 30d` or `--period 90d` before assuming the data isn't there. The CLI's empty-state messages will usually say this if you read them.
- **Quoting line numbers without reconciling**: `sentry issue view` shows the line where the deployed code threw. Don't quote that to the user before establishing what's in their *local* checkout — they may be on a branch where that file has been refactored or moved. See [references/workflows/investigate-an-issue.md](references/workflows/investigate-an-issue.md) for the reconciliation checklist.

### Aliases worth knowing

- `sentry issues` → `sentry issue list` (plural alias on the most common command)
- `sentry issue ls` → `sentry issue list`
- `-t` → `--period` (most list commands)
- `-w` → `--web` (open the resource in browser, on view commands)
- `-n` → `--limit`, `-q` → `--query`, `-f` → `--fresh`, `-F` → `--field`

## Prerequisites

The CLI must be installed and authenticated before use.

### Installation

```bash
curl https://cli.sentry.dev/install -fsS | bash
curl https://cli.sentry.dev/install -fsS | bash -s -- --version nightly

# Or install via npm/pnpm/bun
npm install -g sentry
```

### Authentication

Quick commands:

```bash
sentry auth login                              # interactive OAuth (broad default scopes)
sentry auth login --token YOUR_SENTRY_API_TOKEN # use a manually-scoped token instead
sentry auth status
sentry auth status --json                      # includes source (oauth vs token), expiry
sentry auth whoami                             # identity behind the current token
sentry auth logout
```

To see what scopes the active token actually has:

```bash
sentry api /api/0/ --json | jq -r '.auth.scopes[]'
```

#### Recommended token scopes (least privilege)

Best practice is to run this skill with a **User Auth Token** scoped to only what investigation needs:

| Scope | Why needed |
|---|---|
| `org:read` | List and view orgs |
| `project:read` | List and view projects (**not** `project:admin`, which would allow deletion) |
| `event:read` | Read issues, events, stack traces |
| `event:write` | The only write scope — bounded to issue status mutations (resolve, unresolve, archive). Does **not** allow issue deletion (that needs `event:admin`). |
| `team:read` | Team listing and assignee info |
| `member:read` | Assignee/user info on issues |

**Pointedly excluded**: `project:admin`, `project:write`, `event:admin`, `org:write`, `org:admin`, `team:write`, `team:admin`, `member:write`, `member:admin`, `org:ci`, `project:releases`. The two CLI commands blocked by frontmatter (`sentry project delete`, `sentry trial start`) need `project:admin` and `org:admin` respectively — so if the token lacks both, the corresponding `sentry api` workarounds also fail at the server.

#### Creating a least-privileged token

1. Visit https://sentry.io/settings/account/api/auth-tokens/
2. Click "Create New Token"
3. Tick **only** the six scopes above
4. `sentry auth login --token <new-token>` to switch to it
5. `sentry auth status` to confirm authentication
6. `sentry api /api/0/ --json | jq -r '.auth.scopes[]'` to verify the scope list matches the recommendation

User Auth Tokens are the right choice. **Not** Organization Auth Tokens (`sntrys_...`) — those have fixed scopes designed for CI/release work and can't be customised.

#### Defence-in-depth model

This skill enforces destructive-action safety at two independent layers:

1. **Frontmatter `disallowed-tools`** — blocks `sentry project delete` and `sentry trial start` at the Claude Code permission level. Those CLI commands cannot run while the skill is active.
2. **Least-privileged token** — even if an agent constructs an equivalent `sentry api` call (e.g. `sentry api /api/0/projects/<org>/<project>/ --method DELETE`), the Sentry server rejects it because the token lacks `project:admin` / `org:admin`.

Both layers should be in place. The frontmatter alone still allows the `sentry api` backdoor. The token alone still allows the CLI command path. Together, the cataclysmic options are double-blocked, while `sentry issue resolve PROJX-1234` and similar low-stakes mutations remain available — which is exactly what the lazydev "can you mark that resolved?" flow needs.

The first-action scope check earlier in this skill makes the second layer visible — if the token in use is broader than the recommendation, the agent warns the user before doing real work.

## Command Reference

### Auth

Authenticate with Sentry

- `sentry auth login` — Authenticate with Sentry
- `sentry auth logout` — Log out of Sentry
- `sentry auth refresh` — Refresh your authentication token
- `sentry auth status` — View authentication status
- `sentry auth token` — Print the stored authentication token
- `sentry auth whoami` — Show the currently authenticated identity

→ Full flags and examples: `references/auth.md`

### Org

Work with Sentry organizations

- `sentry org list` — List organizations
- `sentry org view <org>` — View details of an organization

→ Full flags and examples: `references/org.md`

### Project

Work with Sentry projects

- `sentry project create <name> <platform>` — Create a new project
- `sentry project delete <org/project>` — Delete a project
- `sentry project list <org/project>` — List projects
- `sentry project view <org/project>` — View details of a project

→ Full flags and examples: `references/project.md`

### Issue

Manage Sentry issues

- `sentry issue list <org/project>` — List issues in a project
- `sentry issue events <issue>` — List events for a specific issue
- `sentry issue explain <issue>` — Analyze an issue's root cause using Seer AI
- `sentry issue plan <issue>` — Generate a solution plan using Seer AI
- `sentry issue view <issue>` — View details of a specific issue
- `sentry issue resolve <issue>` — Mark an issue as resolved
- `sentry issue unresolve <issue>` — Reopen a resolved issue
- `sentry issue archive <issue>` — Archive (ignore) an issue
- `sentry issue merge <issue...>` — Merge 2+ issues into a single canonical group

→ Full flags and examples: `references/issue.md`

### Event

View and list Sentry events

- `sentry event view <org/project/event-id...>` — View details of one or more events
- `sentry event list <issue>` — List events for an issue

→ Full flags and examples: `references/event.md`

### API

Make an authenticated API request

- `sentry api <endpoint>` — Make an authenticated API request

→ Full flags and examples: `references/api.md`

### CLI

CLI-related commands

- `sentry cli defaults <key value...>` — View and manage default settings
- `sentry cli feedback <message...>` — Send feedback about the CLI
- `sentry cli fix` — Diagnose and repair CLI database issues
- `sentry cli setup` — Configure shell integration
- `sentry cli upgrade <version>` — Update the Sentry CLI to the latest version

→ Full flags and examples: `references/cli.md`

### Dashboard

Manage Sentry dashboards

- `sentry dashboard list <org/title-filter...>` — List dashboards
- `sentry dashboard view <org/project/dashboard...>` — View a dashboard
- `sentry dashboard create <org/project/title...>` — Create a dashboard
- `sentry dashboard widget add <org/project/dashboard/title...>` — Add a widget to a dashboard
- `sentry dashboard widget edit <org/project/dashboard...>` — Edit a widget in a dashboard
- `sentry dashboard widget delete <org/project/dashboard...>` — Delete a widget from a dashboard
- `sentry dashboard revisions <org/dashboard...>` — List dashboard revisions
- `sentry dashboard restore <org/dashboard...>` — Restore a dashboard revision

→ Full flags and examples: `references/dashboard.md`

### Replay

Search and inspect Session Replays

- `sentry replay list <org/project>` — List recent Session Replays
- `sentry replay view <replay-id-or-url...>` — View a Session Replay

→ Full flags and examples: `references/replay.md`

### Release

Work with Sentry releases

- `sentry release list <org/project>` — List releases with adoption and health metrics
- `sentry release view <org/version...>` — View release details with health metrics
- `sentry release create <org/version...>` — Create a release
- `sentry release finalize <org/version...>` — Finalize a release
- `sentry release delete <org/version...>` — Delete a release
- `sentry release deploy <org/version environment name...>` — Create a deploy for a release
- `sentry release deploys <org/version...>` — List deploys for a release
- `sentry release set-commits <org/version...>` — Set commits for a release
- `sentry release propose-version` — Propose a release version

→ Full flags and examples: `references/release.md`

### Repo

Work with Sentry repositories

- `sentry repo list <org/project>` — List repositories

→ Full flags and examples: `references/repo.md`

### Team

Work with Sentry teams

- `sentry team list <org/project>` — List teams

→ Full flags and examples: `references/team.md`

### Explore

Query aggregate event data (Explore)

- `sentry explore <target>` — Query aggregate event data (Explore)

→ Full flags and examples: `references/explore.md`

### Log

View Sentry logs

- `sentry log list <org/project-or-trace-id...>` — List logs from a project
- `sentry log view <org/project/log-id...>` — View details of one or more log entries

→ Full flags and examples: `references/log.md`

### Sourcemap

Manage sourcemaps

- `sentry sourcemap inject <directory>` — Inject debug IDs into JavaScript files and sourcemaps
- `sentry sourcemap upload <directory>` — Upload sourcemaps to Sentry

→ Full flags and examples: `references/sourcemap.md`

### Span

List and view spans in projects or traces

- `sentry span list <org/project/trace-id...>` — List spans in a project or trace
- `sentry span view <trace-id/span-id...>` — View details of specific spans

→ Full flags and examples: `references/span.md`

### Trace

View distributed traces

- `sentry trace list <org/project>` — List recent traces in a project
- `sentry trace view <org/project/trace-id...>` — View details of a specific trace
- `sentry trace logs <org/project/trace-id...>` — View logs associated with a trace

→ Full flags and examples: `references/trace.md`

### Trial

Manage product trials

- `sentry trial list <org>` — List product trials
- `sentry trial start <name> <org>` — Start a product trial

→ Full flags and examples: `references/trial.md`

### Init

Initialize Sentry in your project (experimental)

- `sentry init <target> <directory>` — Initialize Sentry in your project (experimental)

→ Full flags and examples: `references/init.md`

### Schema

Browse the Sentry API schema

- `sentry schema <resource...>` — Browse the Sentry API schema

→ Full flags and examples: `references/schema.md`

## Global Options

All commands support the following global options:

- `--help` - Show help for the command
- `--version` - Show CLI version
- `--log-level <level>` - Set log verbosity (`error`, `warn`, `log`, `info`, `debug`, `trace`). Overrides `SENTRY_LOG_LEVEL`
- `--verbose` - Shorthand for `--log-level debug`

## Output Formats

### JSON Output

Most list and view commands support `--json` flag for JSON output, making it easy to integrate with other tools:

```bash
sentry org list --json | jq '.[] | .slug'
```

### Opening in Browser

View commands support `-w` or `--web` flag to open the resource in your browser:

```bash
sentry issue view PROJ-123 -w
```
