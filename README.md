# claude-sentry

A [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skill for investigating Sentry issues from the command line. It targets the **new** Go-based [`sentry`](https://cli.sentry.dev) CLI — *not* the older Rust `sentry-cli` tool, which is a separate codebase in maintenance mode.

The skill turns the "I just got a Sentry alert at 11pm and have no idea where to start" workflow into a tight, opinionated investigation: pin the local checkout's state loudly, reconcile against deployed code, compare the developer's read against Sentry's Seer AI, and produce a verdict in under thirty lines without ever pasting a raw stack trace.

## What it does well

- **End-to-end issue investigation** — "can you look into `PROJX-1234`?" → cross-references the Sentry data with the local checkout, flags drift, and reports back in plain English
- **Senior-dev overviews** — "what's been cropping up this week across the org?"
- **Lightweight mutations** — "mark that one resolved", "archive this one" — these stay available
- **Trace, span, log, and Explore queries** — the full read side of the Sentry API

## Installation

Requirements on the developer's machine:

- The new Sentry CLI:

  ```bash
  curl https://cli.sentry.dev/install -fsS | bash
  sentry auth login
  ```

- `jq` for JSON filtering. The skill works without it (a `--fields`-based fallback is documented inline), but `jq` is the clean path. Already on most dev machines; otherwise `brew install jq` / `apt install jq`.

- Claude Code, with this repo's skill discoverable under `~/.claude/skills/`:

  ```bash
  git clone https://github.com/<your-user>/claude-sentry.git
  ln -s "$(pwd)/claude-sentry/skills/sentry-cli" ~/.claude/skills/sentry-cli
  ```

  (Or copy the directory across instead of symlinking — symlink is easier to keep updated as the repo evolves.)

## Recommended token scopes

The skill works best with a least-privileged User Auth Token granted **only** these six scopes:

| Scope | Purpose |
|---|---|
| `org:read` | List and view organisations |
| `project:read` | List and view projects |
| `event:read` | Read issues, events, stack traces |
| `event:write` | Mark issues resolved / unresolved / archived (the *only* write scope needed) |
| `team:read` | Team listing and assignee info |
| `member:read` | Assignee / user info on issues |

Notably excluded: `project:admin`, `project:write`, `event:admin`, `org:write`, `org:admin`, `org:ci`, `project:releases`, and the `team:*` / `member:*` write/admin scopes. Create the token at <https://sentry.io/settings/account/api/auth-tokens/> and tick only the six above. Use it with `sentry auth login --token <token>`.

The skill runs a check on first use of every session and warns if the active token is broader than the recommended set. The reasoning lives in [`skills/sentry-cli/SKILL.md`](skills/sentry-cli/SKILL.md) under the *First action: check the token's scopes* and *Authentication* sections.

## Safety measures

Beyond least-privilege auth:

1. **Frontmatter `disallowed-tools`** scopes the destructive denials to Claude Code's permission layer — those CLI commands cannot run while the skill is active, regardless of what the model decides.
2. **First-action token-scope warning** — surfaces over-broad tokens to the developer *before* the agent does any real work, so an admin-scoped token is at least an informed choice rather than an accident.
3. **Read-only git rule** is documented as a hard line, with the rationale explained, so the agent has a *why* to defer to rather than just a rule to ignore.
4. **No raw output dumping** — the playbook's output shape is verdict-first, capped at roughly thirty lines, with stack traces and full event JSON saved for follow-up requests.

The two layers together mean that the destructive endpoints are double-blocked (CLI command at one layer, API method at the other), while the lazydev "yep, mark that resolved" flow still works through both.

## How it matches Sentry issues to local code

This was the trickiest design problem and probably the most useful part of the skill.

Sentry projects and local checkouts don't map cleanly:

- **Code mappings can be auto-generated and wrong.** During development we found a Sentry project mapped to two GitHub repos at once, one of which belonged to a completely unrelated project. Sentry's UI happily picks whichever it likes.
- **Sentry's Seer AI ("issue explain") reaches beyond code mappings.** It content-searches the org's repos and can attribute an issue to a repo that isn't even in the project's mapping list.
- **Local checkouts often disagree with deployed code.** Different directory name from the repo name; on a feature branch; eight months behind; partway through a refactor.

So the skill:

1. Runs `sentry api /api/0/projects/<org>/<project>/seer/preferences/` to read the *actual* code-mapping repos for the project.
2. If multiple repos appear (the common case for misconfigured projects), it asks the developer which is correct — it does **not** guess by name similarity.
3. Searches `~/Documents/code`, `~/code`, and `~/projects` for the local checkout, accepting that the directory name may not match the repo name.
4. Reports the checkout's path, branch, last-commit age, and dirty-tree state *up-front, before quoting any line numbers*, so the developer can spot a "haven't pulled this in eight months" situation immediately.
5. Reads from `git show HEAD:<path>` or a pre-existing `origin/<default-branch>:<path>` ref when the working tree is dirty — never running `git fetch` to bring refs up to date.
6. Filters stack frames using Sentry's own `[in-app]` markers rather than maintaining a regex of vendor paths.
7. Compares its own read of the bug to Sentry Seer's `issue explain` output, surfacing genuine disagreement as a product question for the developer rather than parroting Sentry.

The hero playbook lives at [`skills/sentry-cli/references/workflows/investigate-an-issue.md`](skills/sentry-cli/references/workflows/investigate-an-issue.md), with a worked (sanitised) example demonstrating the output shape.

## Layout

```
skills/sentry-cli/
├── SKILL.md                              ← entry point Claude Code loads
├── references/
│   ├── workflows/
│   │   └── investigate-an-issue.md       ← the hero playbook
│   ├── api.md, auth.md, ...              ← per-command reference docs
│   └── ...
```

The per-command files under `references/` were generated from `sentry --help` output and serve as a quick lookup. The playbook under `references/workflows/` is the considered design document — that's where the opinions live.

## License

MIT — see [LICENSE](LICENSE).
