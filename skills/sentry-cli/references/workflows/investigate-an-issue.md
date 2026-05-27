# Investigate an Issue End-to-End

Use this playbook when the user pastes an issue ID or link — e.g. *"can you look into PROJX-1234"*, *"what's going on with that sentry alert"*, or just a Sentry URL. The target output is a tight, actionable verdict the developer can read in under 30 seconds — **not** a wall of stack-trace.

You bring three things Sentry alone cannot:

1. **Knowledge of the local code** — what's actually checked out on this machine, on which branch, how stale.
2. **An independent read** to compare against Seer's hypothesis, rather than parroting it.
3. **Plain-English framing** that distinguishes *what's broken* (code) from *what should happen* (product).

> ### ⛔ Read-only git only
>
> Inspection commands are fine: `git log`, `git show`, `git status`, `git diff`, `git branch`, `git remote -v`, `git rev-parse`, `git blame`.
>
> **Never run** `git fetch`, `git pull`, `git checkout`, `git stash`, `git reset`, `git merge`, `git rebase`, or anything else that touches refs or the working tree. The developer is probably mid-feature in *some other* project and just got a Sentry alert for *this* one; surprise remote output or a working-tree change is the last thing they need. If a ref doesn't exist locally (e.g. you'd like to inspect `origin/main` but it was never fetched), work without it — don't fetch.

## Pre-flight: pin the local checkout LOUDLY

Skipping this step is the #1 way to mislead the developer.

### 1. Get Sentry's claim

```bash
sentry issue view <SHORT-ID>
```

Note: project slug, claimed file path(s) in the culprit and stack trace, first-seen / last-seen / event count.

**Heads-up on output size:** issues with long stack traces produce 50–100KB of output, and the Bash tool will usually spill it to a tool-results file with a preview. When that happens, read the saved file in slices using the Read tool's `offset`/`limit` (rather than re-running `issue view`). The `[in-app]` markers tell you which frames matter — you can skip past blocks of vendor frames without reading them.

### 2. Verify which repo Sentry thinks this is

```bash
sentry api /api/0/projects/<org>/<project>/seer/preferences/ --json \
  | jq '.code_mapping_repos[].name'
```

If **multiple repos** appear, *ask the user which is correct* — don't guess by name similarity. Auto-generated code mappings can attach the wrong repo to a project (especially when several apps share the same path layout). Seer's `issue explain` can also reach beyond code-mappings via content search, and will sometimes attribute an issue to a repo that isn't even in the mapping list. Treat the repo attribution as a hint.

### 3. Find the local checkout

Common starting points:

- The current working directory (`pwd`)
- `~/Documents/code/<project>`, `~/code/<project>`, `~/projects/<project>`
- Search if the directory name might differ from the repo name:

```bash
find ~/Documents/code -maxdepth 3 -type d -iname '*<keyword>*' 2>/dev/null
```

If nothing comes up, ask the user where the project lives. Do **not** attempt to clone unprompted.

### 4. Report the local checkout state up-front

Before any line-number quotes appear in your reply, state:

- **Path** of the checkout
- **Branch** currently checked out
- **Age** of the last commit (`git log -1 --format='%h %s (%cr)'`)
- **Working tree clean?** (`git status --short`)
- **Remote matches Sentry's mapping?** (`git remote -v`)

Example (fictional):

> Local checkout: `~/Documents/code/projx`, branch `feature/refactor-auth`, last commit 8 months ago, working tree clean. Remote `acme-corp/projx` ≈ matches Sentry's `AcmeCorp/projx` (slight org-name mismatch — assumed to be the same repo). **Note: 8-month-old branch may not reflect what's currently deployed.** Findings below are reconciled against this snapshot.

Stale-by-months local clones are *common* (a senior dev covering for someone on leave, a junior inheriting a project, a service that just hasn't shipped in a while). Even when file paths line up, **call out the age** so the developer can decide whether to update their local before acting on your suggestions. Do not run `git pull` or `git fetch` yourself.

## Filter signal from noise

`sentry issue view` annotates app frames with `[in-app]`. Use that flag — don't regex-filter `/vendor/` paths or maintain your own allowlist.

Walk the in-app frames in **call order** (outermost → bug site) and surface them as a brief call chain in plain prose, not as a pasted trace.

## Reconcile against the local code

For each in-app frame:

- Does the file exist locally?
- Does the symbol (method, function, closure) exist? `grep -n '<symbol>' <file>` is usually enough.
- Are line numbers within ~10 lines of Sentry's? That's "structurally aligned". 50+ lines off, or symbol missing → deployed and local have meaningfully diverged.

Then check the git history of the affected files:

```bash
git log --oneline -10 -- <path>
```

Past commits often explain *why* a bug persists. A message like *"Put exception handling around the failing call"* tells you the error is **known and swallowed**, not new — which usually changes the verdict from "fix this" to "fix this *and* consider removing the swallowing try/catch".

## Compare Seer's hypothesis to your own reading (don't skip this step)

Once you have an independent reading of the bug, **always** run:

```bash
sentry issue explain <SHORT-ID>  # 10-60 seconds typically
```

Don't skip this because you think you already know the answer — Seer's view is the cross-check that catches the cases where *you're* wrong, and it's part of the value-add the developer is asking for. The output is short (often one sentence). **Do not paraphrase it.** Compare:

- Where do you and Seer agree? That's the strongest signal.
- Where do you differ? Often Seer's framing is correct at the *what's broken* level but assumes a different *what should happen*. Surface that divergence as a **product question** for the developer rather than a code question.

The "Seer's hypothesis" and "My reading and where it differs" sections of the output shape (below) both depend on having run this — there is no good shortcut.

If a fix plan would also help (the developer asked for one, or your reading has multiple branches that could each be expanded):

```bash
sentry issue plan <SHORT-ID>
```

Same treatment — integrate Seer's plan into your own reading rather than pasting it.

## Volume math reframes scary numbers

Divide event count by days-since-first-seen.

> 5648 events × first seen 2024-11-07 → ≈ 10/day → small stable trickle, not a fire.

Include this rate in your verdict. It often changes the urgency framing dramatically (a "5000-event issue" with a 6-event-per-day rate is very different from a "5000-event issue" with a 5000-event-per-hour rate).

## Output shape

Aim for under 30 lines. Suggested structure:

> **Reconciliation:** path, branch, age, staleness warning if relevant.
>
> **What's happening:** one or two sentences, with the in-app call chain in plain text.
>
> **Why it's still firing:** if relevant — masking try/catches, ignored cron output, unhandled retries (usually found via `git log`).
>
> **Seer's hypothesis:** one line.
>
> **My reading and where it differs:** one or two lines, with the product question surfaced if applicable.
>
> **Volume:** rate per day, total since first seen.
>
> **Fix options:** 2–4 ranked options, briefly. Lead with the smallest blast radius.
>
> **One thing to mention:** optional final note — e.g. a swallowing try/catch worth narrowing once the underlying bug is fixed.

Do **not** paste the raw stack trace, full event JSON, or all 100 fields of `issue view` output unless the developer asks. Save those for follow-up requests.

## Decision branches

**Multiple code-mapping repos appear**
Ask the user which is correct before pulling a local checkout. Don't pick by name similarity.

**Local checkout missing**
`find` siblings in common code dirs. If still nothing, ask where the project lives or to clone it. Don't clone unprompted.

**Local checkout very stale** (last commit > 6 months ago)
Proceed, but flag the staleness *loudly* in your reply before any line-number claims. Do **not** run `git fetch` or `git pull` to "freshen up" — that's the developer's call, not yours. If `origin/<default-branch>` already exists locally (from a prior fetch) and looks more current than the checked-out branch, prefer reading from it (`git show origin/main:<path>`) — but only use refs that already exist; don't fetch new ones.

**Working tree dirty**
Note it briefly. The developer might be mid-refactor of the affected file. If the affected file is among the modified ones, mention that explicitly and read from `HEAD` (`git show HEAD:<path>`) or from `origin/<default-branch>` *if that ref already exists locally*, rather than from the working tree. Never run `git stash` or `git checkout` to clean up.

**File or symbol not found in local checkout**
Check whether it was renamed/moved: `git log --all --oneline -- <path>` and `grep -rn '<symbol>' .` from the repo root. If still missing, deployed code has truly diverged — *report this*, do not fabricate a guess.

**Sentry path uses old layout but local has modernised** (e.g. Sentry says `app/User.php` but local has `app/Models/User.php`)
Partial Laravel/framework modernisation. Report what you find at *both* paths and let the developer say which represents deployed.

**Seer attributes a repo not in the code-mapping list**
Seer can content-search the org's repos. The named repo may genuinely be relevant (e.g. it configures a URL or job that the buggy project consumes) — or may be a red herring. Surface Seer's claim, but verify against the project's actual code-mapping repos before pulling a checkout.

## Worked example (illustrative — names are fictional)

An illustrative run against `PROJX-1234` (an FK-constraint violation in a Laravel nightly cron). The *shape* below — what to say, in what order — is the target. The specific findings, table names, and method names will obviously differ per issue. If you find yourself investigating a real issue and notice the headline pattern looks oddly similar to this example, re-derive everything from the actual data rather than relying on the numbers here.

> **Reconciliation:** `~/Documents/code/projx`, branch `feature/refactor-auth`, last commit 8 months ago, working tree clean. **8-month-old local — may not reflect deployed code.** Paths and method names match Sentry within 5 lines, reconciliation is trustworthy *for this snapshot*.
>
> **What's happening:** the daily `PruneStaleUsers` Artisan command iterates all users and calls `$user->deleteIfDormant()`. For users with rows in `user_widget_assignments`, the FK on `users.id` blocks the delete and MySQL throws SQLSTATE[23000].
>
> **Why it's still firing 18 months later:** commit `a1b2c3d4` deliberately wrapped the call in `try/catch` and routes to `\Sentry\captureException`. The error is caught — so the cron keeps working — but the underlying bug was never fixed.
>
> **Seer's hypothesis:** fix `isDormant()` to also account for widget-assignment records.
>
> **My reading and where it differs:** Seer's framing is sound *if* widget assignments are meaningful historical activity that should keep the user alive. The alternative is "widget assignments are disposable; cascade-delete them with the user". The method name `isDormant` and the existence of an entire pivot table suggest the assignments *are* meaningful activity, so I'd lean Seer's way — but that's a product call, not a code call.
>
> **Volume:** ≈10/day, stable for ~18 months. Small steady trickle, not a fire.
>
> **Fix options (small → large blast radius):**
> 1. Extend `isDormant()` to also check `widgetAssignments()->exists()`. Smallest, matches intent.
> 2. `deleting` model event on `User` to cascade related rows. Heavier — affects every delete of a user, not just the prune path.
> 3. Schema-level `ON DELETE CASCADE` migration on the FK. Heaviest, but catches every code path that ever deletes a user.
>
> **One thing to mention:** the try/catch is doing its job *too* well — masking the bug while still cluttering Sentry. After fixing, consider narrowing or removing it so future FK failures here become loud bugs rather than ~10-a-day noise.

That entire report ran without ever pasting the stack trace, ran one `sentry api` call to disambiguate the code mapping, and ran one `sentry issue explain` for Seer's view. Keep your workflow that lean.
