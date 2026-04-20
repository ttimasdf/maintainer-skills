---
name: upstream-sync
description: Sync a GitLab fork with the latest tagged release from an upstream repository. Fetches the newest upstream tag, merges it into the current development branch via a dedicated sync branch, resolves conflicts intelligently, tracks the last-merged version in a `.upstream-version` dotfile, and opens a GitLab merge request. Upstream URL may be passed as an argument or inferred from the existing `upstream` git remote. Use this skill whenever the user wants to update a fork, pull upstream changes, sync with upstream, catch up on an upstream release, or invokes `/upstream-sync` (with or without a URL) from CI. Designed to run unattended inside GitLab CI jobs but works interactively too.
---

# Upstream Sync

Keep a long-lived GitLab fork in sync with tagged releases from an upstream repository. Runs unattended in CI (`claude -p "/upstream-sync [UPSTREAM_URL]"`) or interactively.

## What this skill does

Using an upstream repo URL (from the skill argument, or inferred from the git remote named `upstream`), the skill:

1. Determines the target development branch (the currently checked-out branch on `origin`).
2. Fetches the latest semver-style tag from upstream.
3. Compares it against `.upstream-version` (the dotfile tracking the last successfully merged upstream tag).
4. If there's a new tag: creates a sync branch, merges the upstream tag, resolves conflicts intelligently, updates `.upstream-version`, pushes, and opens a merge request back to the dev branch.
5. If already up to date: exits cleanly with a short message and zero exit code.

## Inputs and invariants

- **Argument 1 (optional)**: upstream repo URL (HTTPS or SSH). Resolution order:
  - If the arg is present, use it. If a remote named `upstream` already exists with a different URL, update it with `git remote set-url`; if no such remote exists, add it.
  - If the arg is absent, read the URL from the existing `upstream` remote (`git remote get-url upstream`). This is the common CI case for forks that already have the remote configured — `claude -p /upstream-sync` with no args just works.
  - If the arg is absent and no `upstream` remote exists, abort with a clear error message: "no upstream URL provided and no 'upstream' git remote configured".
- **Current working directory**: a checked-out clone of the fork. `HEAD` is on the development branch that MRs should target.
- **Tracking file**: `.upstream-version` at the repo root. First line is the last-merged upstream tag (e.g. `v1.4.2`). If the file doesn't exist, treat this as a first run — see *First-run bootstrap* below.
- **Auth**: in CI, expect `GITLAB_TOKEN` or `CI_JOB_TOKEN` to be set; locally, expect `glab auth status` to succeed.

Do not modify anything on the current dev branch directly. All work happens on a throwaway `upstream-sync/<tag>` branch.

## Workflow

Run these steps in order. Stop and report clearly if any step fails in a way you can't recover from.

### 1. Validate environment

- Confirm you're inside a git repo: `git rev-parse --show-toplevel`. `cd` to that root.
- Read the current branch: `git rev-parse --abbrev-ref HEAD`. Save as `DEV_BRANCH`. If it's `HEAD` (detached), abort — CI should check out a real branch.
- Confirm the working tree is clean (`git status --porcelain` is empty). If dirty, abort — something upstream of this skill is misconfigured.

### 2. Resolve the upstream URL and set up the remote

The URL may come from the skill argument or from an existing `upstream` remote. Pick one:

```bash
ARG_URL="${1:-}"  # whatever was passed to the skill
EXISTING_URL="$(git remote get-url upstream 2>/dev/null || true)"

if [[ -n "$ARG_URL" ]]; then
    UPSTREAM_URL="$ARG_URL"
    if [[ -n "$EXISTING_URL" ]]; then
        # Remote exists — update only if the URL differs.
        if [[ "$EXISTING_URL" != "$UPSTREAM_URL" ]]; then
            git remote set-url upstream "$UPSTREAM_URL"
        fi
    else
        git remote add upstream "$UPSTREAM_URL"
    fi
elif [[ -n "$EXISTING_URL" ]]; then
    UPSTREAM_URL="$EXISTING_URL"
else
    echo "Error: no upstream URL provided and no 'upstream' git remote configured." >&2
    echo "Pass the URL as the first argument or run 'git remote add upstream <url>' first." >&2
    exit 2
fi

git fetch upstream --tags --prune
```

The arg-wins, remote-as-fallback precedence matters: forks that already have `upstream` configured can just run `/upstream-sync` with no args, while CI pipelines that pass the URL explicitly always win over whatever's in `.git/config` (so you can migrate upstream URLs by changing one variable in CI without touching the repo).

### 3. Find the latest upstream tag

Prefer the highest semver tag, not the most recently created one — upstream projects sometimes push patches to older branches.

```bash
LATEST_TAG=$(git -c versionsort.suffix=-pre \
  for-each-ref --sort=-v:refname --format='%(refname:short)' \
  'refs/tags/v[0-9]*' 'refs/tags/[0-9]*' \
  --count=1)
```

Note the `v[0-9]*` glob, not `v*`. Upstream repos often publish auxiliary tags with a prefix (e.g. `vscode-v0.0.13`, `github-v1.2.3`, `latest`) alongside the main product tags. `v*` would match those and, because version sort ranks them strangely, sometimes returns them as "highest." `v[0-9]*` requires a digit immediately after the `v`, which reliably isolates the main product's semver tags. If a particular upstream uses a different scheme (e.g. release tags prefixed `release-v...`), override the glob inline.

If that returns nothing, fall back to `git describe --tags $(git rev-list --tags --max-count=1)`. If still nothing, report "upstream has no tags" and exit 0 — a tagless upstream isn't an error, just nothing to sync.

Filter out pre-release tags (anything containing `-alpha`, `-beta`, `-rc`, `-pre`) unless the `.upstream-version` itself points at a pre-release — in that case the user has opted into them.

### 4. Compare against `.upstream-version`

- If `.upstream-version` exists, read its first line as `LAST_TAG`.
- If `.upstream-version` doesn't exist, this is a **first-run bootstrap** — see *First-run bootstrap* below. After bootstrap, `LAST_TAG` is set to whatever baseline the bootstrap discovered, and execution continues with the comparison below as if `.upstream-version` had been there all along.
- If `LAST_TAG == LATEST_TAG`: print `Already at latest upstream tag: $LATEST_TAG` and exit 0. This is the common case in periodic CI.
- Otherwise, fall through to step 5 to merge `LAST_TAG..LATEST_TAG`.

#### First-run bootstrap

The fork's history almost certainly already contains some upstream commit (it was forked from upstream at some point). The bootstrap's job is to **discover which upstream tag the fork is currently sitting on**, not to invent a baseline by importing the latest tag.

Iterate upstream's main-product tags from newest to oldest and pick the first one whose commit is reachable from the current dev branch. The result is a `LAST_TAG` variable that the rest of the workflow uses exactly as if `.upstream-version` had contained it on disk all along — no intermediate commit, no separate "bootstrap branch":

```bash
LAST_TAG=""
while read -r tag; do
    [[ -z "$tag" ]] && continue
    if git merge-base --is-ancestor "refs/tags/$tag" HEAD 2>/dev/null; then
        LAST_TAG="$tag"
        break
    fi
done < <(git -c versionsort.suffix=-pre \
    for-each-ref --sort=-v:refname --format='%(refname:short)' \
    'refs/tags/v[0-9]*' 'refs/tags/[0-9]*')
BOOTSTRAPPED=1   # remember this so the MR description can explain what happened
```

Two outcomes:

- **A baseline tag was found** (the common case). Continue to step 4's comparison with that `LAST_TAG`. If it equals `LATEST_TAG`, the workflow proceeds as the normal "no-op except this is the first time we're recording it": create a sync branch, write `.upstream-version`, commit, push, open a tiny MR titled `chore: bootstrap upstream tracking at <tag>`. If `LATEST_TAG` is newer, the workflow does a normal merge from the discovered baseline forward — the resulting MR contains both the merge commit *and* the new `.upstream-version` file, naturally. Either way, only one MR is produced.
- **No baseline tag was found** (no upstream tag is in the fork's ancestry — the fork was re-initialized, or has been heavily rewritten and lost upstream's history). Don't guess. Set `LAST_TAG="$LATEST_TAG"` so the comparison in step 4 reports "already up to date" and the bootstrap MR is just the `.upstream-version` file pinned to the current latest. **Prominently** flag in the MR description that no shared history was detected and ask the human to verify this baseline is sensible — future syncs will only kick in once a *newer* tag ships, so a wrong baseline here means future merges silently miss everything between the real baseline and `LATEST_TAG`.

When `BOOTSTRAPPED=1`, the MR description (step 9) should open with a short paragraph stating that this is a first-run bootstrap and which baseline tag was discovered (or that none was found, in the no-shared-history case). This is the only thing reviewers need to know to trust the rest of the MR.

Why discover-then-fall-through instead of just bootstrapping to "latest"? A naive bootstrap to the latest tag would silently lie about the fork's actual baseline. The next sync would then look at `LAST_TAG..NEW_TAG` and miss the entire range from the *real* baseline to the wrongly-recorded one — meaning the first real sync would be huge and the conflict-resolution log would mix many releases together. Discovering the true baseline keeps every future sync small and traceable.

### 5. Create the sync branch

```bash
SYNC_BRANCH="upstream-sync/$LATEST_TAG"
git checkout -b "$SYNC_BRANCH"
```

If the branch already exists locally or on `origin`, it's likely a leftover from a prior failed run. Delete the local copy and recreate; if the remote copy exists, the MR likely already exists too — check via `glab mr list --source-branch "$SYNC_BRANCH"` and either reuse it (force-push) or rename the local branch with a `-retry` suffix. Prefer reusing.

### 6. Merge the upstream tag

```bash
git merge --no-ff --no-edit "refs/tags/$LATEST_TAG" \
  -m "Merge upstream tag $LATEST_TAG into $DEV_BRANCH"
```

If this fails because the histories are unrelated (fork was re-initialised at some point), retry with `--allow-unrelated-histories` — but only if `.upstream-version` doesn't exist (i.e. first-ever merge). Otherwise it's a real problem; abort and flag it.

### 7. Resolve conflicts intelligently

If the merge reports conflicts (`git status` shows `UU`/`AA`/`DU`/`UD` entries), do not blindly accept one side. For each conflicted file:

**Read the file and understand what each side is doing.** Use `git log --oneline $LAST_TAG..HEAD -- <path>` to see fork-side changes and `git log --oneline $LAST_TAG..$LATEST_TAG -- <path>` to see upstream changes. These are the two changesets you're reconciling.

Then apply judgment:

- **Pure additions on both sides (e.g. two new entries in a list)**: keep both.
- **Upstream refactored something the fork also edited**: prefer upstream's structure, port the fork's intent into the new structure. The fork's edits usually express a policy or feature that should survive the refactor; if that's not obvious from the diff, leave a `// TODO(upstream-sync):` comment noting the uncertainty for the human reviewer.
- **Upstream deleted a file the fork modified**: lean toward deleting (upstream's direction wins for lifecycle decisions), but note it prominently in the MR description so the reviewer can push back.
- **Upstream renamed a file the fork modified**: apply the rename, carry the fork's edits forward into the renamed file.
- **Lockfiles / generated files** (`package-lock.json`, `yarn.lock`, `Cargo.lock`, `go.sum`, `poetry.lock`, etc.): don't hand-merge. Take upstream's version, then regenerate by running the project's install command (`npm install`, `cargo build`, `go mod tidy`, etc. — pick the one that matches the repo). If no install command is obvious, take upstream and note it in the MR.
- **Formatting-only conflicts** (whitespace, import order): take upstream.

After each resolution, verify the file parses / at least reads sensibly. `git add` it. When all conflicts are resolved, `git commit --no-edit` to finalize the merge.

Keep a running log of how each conflict was resolved — this becomes the MR description.

### 8. Update the tracking file

```bash
echo "$LATEST_TAG" > .upstream-version
git add .upstream-version
git commit -m "chore: record upstream sync to $LATEST_TAG"
```

Keep this as a separate commit from the merge — it makes the MR diff easier to read and lets reviewers see the bookkeeping change distinctly.

### 9. Push and open the MR

```bash
git push --set-upstream origin "$SYNC_BRANCH" --force-with-lease
```

Then open the MR. Prefer `glab`:

```bash
glab mr create \
  --source-branch "$SYNC_BRANCH" \
  --target-branch "$DEV_BRANCH" \
  --title "Sync upstream: $LATEST_TAG" \
  --description "$(cat /tmp/mr-body.md)" \
  --remove-source-branch \
  --yes
```

If `glab` is unavailable, fall back to the GitLab REST API via `curl` using `$CI_PROJECT_ID` and `$GITLAB_TOKEN` (or `$CI_JOB_TOKEN`). The API path is `POST /projects/:id/merge_requests`.

**MR description contents** (write to `/tmp/mr-body.md` first):

```
## Upstream sync: <LAST_TAG> → <LATEST_TAG>

Automated sync of upstream tag `<LATEST_TAG>` into `<DEV_BRANCH>`.

### Upstream changelog
<link to upstream tag or release notes if derivable from the URL>

### Conflicts resolved
<bullet list from step 7's running log; if no conflicts, say "Clean merge, no conflicts.">

### Reviewer checklist
- [ ] Conflict resolutions preserve fork-specific behavior
- [ ] CI passes
- [ ] `.upstream-version` bumped to `<LATEST_TAG>`

Generated by /upstream-sync.
```

### 10. Report

Print a one-line summary to stdout: either `Opened MR !<n>: <title>` or `Already up to date at <tag>`. In CI, this ends up in the job log.

## Dry-run mode

If `UPSTREAM_SYNC_DRY_RUN=1` is set in the environment, do everything through step 8 normally (branch, merge, conflict resolution, tracking-file bump, local commits) but **skip step 9's push and MR creation**. Instead, print the fully-composed MR body to stdout preceded by the line `=== DRY RUN MR BODY ===` and exit 0. This mode exists for local testing and for running the skill against a fixture repo without a real GitLab server — it exercises all the interesting logic without any external side effects.

## Failure modes and what to do

- **Merge fails for a reason other than conflicts** (e.g., upstream history diverged): don't force it. Abort, leave no MR, exit non-zero so the CI job is visibly red. Leave a message explaining what was tried.
- **Conflict that's genuinely unsafe to resolve automatically** (e.g., diverging crypto code, security-sensitive logic): stop, keep the conflict markers in the file, `git add` it anyway, commit with a message flagging this, and say so prominently in the MR description. Humans finish the job. Better to open an MR with flagged conflicts than to silently make a wrong decision.
- **Push rejected**: likely a branch-protection rule on the sync branch namespace. Report and exit non-zero.
- **MR already exists** for this tag: update the existing MR's description rather than opening a duplicate.

## Why the design is this way

- **Tag-based, not branch-based**: tags are stable release points with known quality. Tracking `upstream/main` in a long-lived fork produces a torrent of unreviewable merges. Tags give the human reviewer a discrete, meaningful unit of change.
- **Dotfile for state**: `.upstream-version` lives in the repo so it follows branches and is atomic with the merge commit. No external database, no CI-variable drift, no "which environment is authoritative" question.
- **Always open an MR, never push to dev directly**: even a clean merge deserves a review pass — the reviewer should at least glance at the upstream changelog to spot anything concerning. CI-driven auto-merge would defeat the point.
- **`--force-with-lease`, not `--force`**: if a human has pushed to the sync branch (e.g., fixing a conflict manually), don't stomp on them.
- **Per-tag sync branches**: makes it obvious which MR corresponds to which upstream release, and lets multiple syncs coexist if one gets stuck in review.

## Example sessions

**Explicit URL arg (typical CI use):**

```
$ claude -p "/upstream-sync https://github.com/upstream-org/project.git"

Detected dev branch: main
Using upstream URL from argument; updating existing 'upstream' remote.
Current tracked upstream: v2.3.1
Latest upstream tag: v2.4.0
Creating branch upstream-sync/v2.4.0...
Merging refs/tags/v2.4.0...
Conflicts in: src/config.rs, package-lock.json
Resolving src/config.rs: upstream added a new field; fork's custom defaults preserved.
Resolving package-lock.json: taking upstream, regenerating via npm install.
Commit: merge + .upstream-version bump
Pushed to origin/upstream-sync/v2.4.0
Opened MR !142: Sync upstream: v2.4.0
```

**No arg, reuses existing remote:**

```
$ claude -p "/upstream-sync"

Detected dev branch: main
Using upstream URL from existing 'upstream' remote: https://github.com/upstream-org/project.git
Already at latest upstream tag: v2.4.0
```
