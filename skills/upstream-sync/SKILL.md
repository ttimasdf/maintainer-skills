---
name: upstream-sync
description: Sync a GitLab fork with the latest tagged release from an upstream repository. Fetches the newest upstream tag, merges it into the current development branch via a dedicated sync branch, resolves conflicts with awareness of fork-specific downstream changes, tracks the upstream repository and last-merged version in a `.upstream-version` dotfile, and opens a GitLab merge request with upstream release notes. Upstream URL may be passed as an argument or inferred from the existing `upstream` git remote. Use this skill whenever the user wants to update a fork, pull upstream changes, sync with upstream, catch up on an upstream release, or invokes `/upstream-sync` (with or without a URL) from CI. Designed to run unattended inside GitLab CI jobs but works interactively too.
---

# Upstream Sync

Keep a long-lived GitLab fork in sync with tagged releases from an upstream repository. Runs unattended in CI (`opencode -p "/upstream-sync [UPSTREAM_URL]"`) or interactively.

## What this skill does

Using an upstream repo URL (from the skill argument, or inferred from the git remote named `upstream`), the skill:

1. Determines the target development branch (the currently checked-out branch on `origin`).
2. Fetches the latest semver-style tag from upstream.
3. Compares it against `.upstream-version` (the dotfile tracking the upstream repo URL and last successfully merged upstream tag).
4. If there's a new tag: creates a git worktree for isolation, creates a sync branch inside it, merges the upstream tag, resolves conflicts intelligently while preserving downstream fork changes from `DOWNSTREAM_CHANGES.md`, proposes updates to superseded `DOWNSTREAM_CHANGES.md` entries (with user confirmation), updates `.upstream-version`, pushes, opens a merge request back to the dev branch, comments with upstream release notes, and cleans up the worktree.
5. If already up to date: exits cleanly with a short message and zero exit code.

## Inputs and invariants

- **Argument 1 (optional)**: upstream repo URL (HTTPS or SSH). Resolution order:
  - If the arg is present, use it. If a remote named `upstream` already exists with a different URL, update it with `git remote set-url`; if no such remote exists, add it.
  - If the arg is absent, read the URL from the existing `upstream` remote (`git remote get-url upstream`). This is the common CI case for forks that already have the remote configured — `opencode -p /upstream-sync` with no args just works.
  - If the arg is absent and no `upstream` remote exists, abort with a clear error message: "no upstream URL provided and no 'upstream' git remote configured".
- **Current working directory**: a checked-out clone of the fork. `HEAD` is on the development branch that MRs should target.
- **Tracking file**: `.upstream-version` at the repo root. It uses shell-style key/value lines:
  ```dotenv
  UPSTREAM_REPO=<repo-url.git>
  UPSTREAM_VERSION=<vN.NN.N>
  ```
  If the file doesn't exist, treat this as a first run — see *First-run bootstrap* below. If it exists in the legacy one-line format, treat the first line as `UPSTREAM_VERSION`, infer `UPSTREAM_REPO` from the argument or `upstream` remote, and rewrite the file in the new format during step 8.
- **Downstream change ledger**: `DOWNSTREAM_CHANGES.md` at the repo root, when present, documents intentional fork-only behavior. Read it before resolving conflicts and use it as the source of truth for changes that should survive upstream merges.
- **Auth**: in CI, expect `GITLAB_TOKEN` or `CI_JOB_TOKEN` to be set; locally, expect `glab auth status` to succeed.

Do not modify anything on the current dev branch directly. All work happens inside a **git worktree** on a throwaway `upstream-sync/<tag>` branch. The worktree isolates the sync from the main working tree — the dev branch stays checked out and untouched throughout.

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

Filter out pre-release tags (anything containing `-alpha`, `-beta`, `-rc`, `-pre`) unless `UPSTREAM_VERSION` in `.upstream-version` itself points at a pre-release — in that case the user has opted into them.

### 4. Compare against `.upstream-version`

- If `.upstream-version` exists in the current key/value format, parse `UPSTREAM_REPO` and `UPSTREAM_VERSION`. Read `UPSTREAM_VERSION` as `LAST_TAG`. If `UPSTREAM_REPO` differs from the resolved `UPSTREAM_URL`, prefer the resolved URL from step 2, continue, and record the URL change in the MR description.
- If `.upstream-version` exists in the legacy one-line format, read its first line as `LAST_TAG`, set `LEGACY_UPSTREAM_VERSION=1`, and rewrite it in the current key/value format during step 8.
- If `.upstream-version` doesn't exist, this is a **first-run bootstrap** — see *First-run bootstrap* below. After bootstrap, `LAST_TAG` is set to whatever baseline the bootstrap discovered, and execution continues with the comparison below as if `.upstream-version` had been there all along.
- If `LAST_TAG == LATEST_TAG` and the tracking file is already in the current format with the resolved `UPSTREAM_URL`: print `Already at latest upstream tag: $LATEST_TAG` and exit 0. This is the common case in periodic CI.
- If `LAST_TAG == LATEST_TAG` but the tracking file is missing, legacy-formatted, or records a different `UPSTREAM_REPO`, create the sync branch and open a bookkeeping MR that only rewrites `.upstream-version`; do not merge upstream again.
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

### 5. Create a worktree and the sync branch

All work from this point onward happens inside a git worktree. This keeps the main working tree on `DEV_BRANCH` completely untouched — no stashing, no branch switching, no risk of leaving the repo in a dirty state.

```bash
REPO_ROOT="$(git rev-parse --show-toplevel)"
SYNC_BRANCH="upstream-sync/$LATEST_TAG"
WORKTREE_DIR="$REPO_ROOT/.git-upstream-sync-worktree"

# Remove leftover worktree from a prior failed run, if any.
if [[ -d "$WORKTREE_DIR" ]]; then
    git worktree remove --force "$WORKTREE_DIR" 2>/dev/null || true
fi

git worktree add -b "$SYNC_BRANCH" "$WORKTREE_DIR" HEAD
cd "$WORKTREE_DIR"
```

This creates a new branch `upstream-sync/<tag>` checked out in a separate directory (`.git-upstream-sync-worktree/` at the repo root). The main working tree stays on `DEV_BRANCH`. All subsequent steps (merge, conflict resolution, tracking file update, push) run inside `$WORKTREE_DIR`.

If the branch already exists locally, `git worktree add -b` will fail. In that case it's likely a leftover from a prior failed run — delete the branch and the worktree, then recreate:

```bash
git branch -D "$SYNC_BRANCH" 2>/dev/null || true
git worktree remove --force "$WORKTREE_DIR" 2>/dev/null || true
git worktree add -b "$SYNC_BRANCH" "$WORKTREE_DIR" HEAD
cd "$WORKTREE_DIR"
```

If the branch already exists on `origin`, the MR likely already exists too — check via `glab mr list --source-branch "$SYNC_BRANCH"` and either reuse it (force-push into the existing worktree after `git reset`) or rename the local branch with a `-retry` suffix. Prefer reusing.

### 6. Merge the upstream tag

```bash
git merge --no-ff --no-edit "refs/tags/$LATEST_TAG" \
  -m "Merge upstream tag $LATEST_TAG into $DEV_BRANCH"
```

If this fails because the histories are unrelated (fork was re-initialised at some point), retry with `--allow-unrelated-histories` — but only if `.upstream-version` doesn't exist (i.e. first-ever merge). Otherwise it's a real problem; abort and flag it.

### 7. Resolve conflicts intelligently

If the merge reports conflicts (`git status` shows `UU`/`AA`/`DU`/`UD` entries), do not blindly accept one side. For each conflicted file:

**Read the downstream ledger first.** If `DOWNSTREAM_CHANGES.md` exists, inspect it before resolving any conflict. Search it for the conflicted path, related feature names, configuration keys, and behavior mentioned by the fork-side diff. Treat entries in this file as intentional downstream behavior that should not be overwritten accidentally.

**Read the file and understand what each side is doing.** Use `git log --oneline $LAST_TAG..HEAD -- <path>` to see fork-side changes and `git log --oneline $LAST_TAG..$LATEST_TAG -- <path>` to see upstream changes. Also inspect the actual diffs from both sides (`git diff $LAST_TAG..HEAD -- <path>` and `git diff $LAST_TAG..$LATEST_TAG -- <path>`). These are the two changesets you're reconciling.

Then apply judgment:

- **Pure additions on both sides (e.g. two new entries in a list)**: keep both.
- **Upstream refactored something the fork also edited**: prefer upstream's structure, port the fork's intent into the new structure. The fork's edits usually express a policy or feature that should survive the refactor; if that's not obvious from the diff, leave a `// TODO(upstream-sync):` comment noting the uncertainty for the human reviewer.
- **A downstream-only behavior is documented in `DOWNSTREAM_CHANGES.md`**: preserve that behavior unless upstream now implements the same feature or policy. If upstream implements the same feature, prefer the upstream implementation, remove the redundant fork-only implementation, and add an explicit warning to the MR description explaining which downstream change was superseded by upstream.
- **Both sides implemented the same feature differently**: use the upstream implementation as the canonical version, then verify any downstream-specific configuration/defaults from `DOWNSTREAM_CHANGES.md` are still represented. Record this in the conflict log as `Downstream change superseded by upstream: <summary>` so reviewers know to verify behavior.
- **Upstream deleted a file the fork modified**: lean toward deleting (upstream's direction wins for lifecycle decisions), but note it prominently in the MR description so the reviewer can push back.
- **Upstream renamed a file the fork modified**: apply the rename, carry the fork's edits forward into the renamed file.
- **Lockfiles / generated files** (`package-lock.json`, `yarn.lock`, `Cargo.lock`, `go.sum`, `poetry.lock`, etc.): don't hand-merge. Take upstream's version, then regenerate by running the project's install command (`npm install`, `cargo build`, `go mod tidy`, etc. — pick the one that matches the repo). If no install command is obvious, take upstream and note it in the MR.
- **Formatting-only conflicts** (whitespace, import order): take upstream.

After each resolution, verify the file parses / at least reads sensibly. `git add` it. When all conflicts are resolved, `git commit --no-edit` to finalize the merge.

Keep a running log of how each conflict was resolved — this becomes the MR description. Separate normal conflict notes from **downstream supersession warnings** so the MR can call out cases where upstream replaced a fork-maintained feature.

### 8. Update the tracking file

```bash
cat > .upstream-version <<EOF
UPSTREAM_REPO=$UPSTREAM_URL
UPSTREAM_VERSION=$LATEST_TAG
EOF
git add .upstream-version
git commit -m "chore: record upstream sync to $LATEST_TAG"
```

Keep this as a separate commit from the merge — it makes the MR diff easier to read and lets reviewers see the bookkeeping change distinctly.

### 9. Review and update `DOWNSTREAM_CHANGES.md`

After resolving conflicts and updating the tracking file, scan `DOWNSTREAM_CHANGES.md` for entries that this sync may have superseded or invalidated. This is the step that fulfills the promise made in the `AGENTS.md` Fork Maintenance section: *"When `/upstream-sync` detects that an upstream release implements the same feature or fix as a downstream change, it will update the entry's status to `superseded` and note the upstream version."*

**Skip this step entirely** if `DOWNSTREAM_CHANGES.md` doesn't exist or contains no entries with `Status: active`.

#### Detection

For each `active` entry in `DOWNSTREAM_CHANGES.md`, check whether the upstream changes in `LAST_TAG..LATEST_TAG` affect it:

1. Read the entry's **Scope** paths and **Files affected** list.
2. Check whether upstream touched those files: `git diff --name-only refs/tags/$LAST_TAG..refs/tags/$LATEST_TAG -- <scope-paths>`.
3. If upstream touched those files, inspect the upstream diff to classify the overlap:
   - **Superseded**: upstream now implements the same feature/fix/config that the fork entry describes. The downstream change is redundant. Examples: upstream added the same config flag, upstream fixed the same bug, upstream introduced the same feature natively.
   - **Partially superseded**: upstream implements part of what the downstream change does, but not all. The entry may need its description updated to reflect what's still fork-specific.
   - **Unrelated overlap**: upstream touched the same file but the changes don't overlap with the downstream modification (e.g., upstream added a new function in a different section). No action needed.
4. Also check for **removed entries**: if upstream deleted a file that a downstream entry's scope references, or if the merge in step 6 resolved a conflict by taking upstream's version entirely (removing the downstream code), the downstream change is effectively gone.

Collect all detected entries into a list. If none are found, skip to step 10.

#### User confirmation

Present the findings to the user and ask for confirmation before modifying `DOWNSTREAM_CHANGES.md`. This is critical — the ledger is a human-maintained document and the agent's classification may be wrong. Format the prompt like:

```
The following DOWNSTREAM_CHANGES.md entries may need updating after this upstream sync:

1. [slug-id]: Short description
   → Superseded by upstream (v<LATEST_TAG>): upstream now implements <summary>
   Proposed: Status → superseded, Superseded by upstream → <LATEST_TAG>

2. [slug-id]: Short description
   → Partially superseded: upstream added <summary>, but fork still overrides <detail>
   Proposed: Update "What this changes" to reflect remaining fork-specific behavior

3. [slug-id]: Short description
   → Removed: upstream deleted <file>, downstream code no longer exists after merge
   Proposed: Status → removed

Apply these updates? (all / none / select specific entries)
```

The user may:
- **Confirm all** — apply every proposed update.
- **Confirm a subset** — apply only selected entries (by number or slug-id).
- **Reject all** — leave `DOWNSTREAM_CHANGES.md` unchanged. Any supersession warnings are still noted in the MR description (step 10), just not written back to the ledger.

In CI (non-interactive), do not modify `DOWNSTREAM_CHANGES.md` automatically. Instead, include the proposed updates as a section in the MR description so a human reviewer can apply them manually after review. This prevents an unattended agent from incorrectly classifying entries.

#### Apply confirmed updates

For each entry the user confirmed, edit `DOWNSTREAM_CHANGES.md`:

- **Superseded**: set `Status` to `superseded`, set `Superseded by upstream` to `$LATEST_TAG`. Do not delete the entry — it's kept for history.
- **Partially superseded**: update the `What this changes` description to reflect what's still fork-specific. Optionally update `Scope` or `Files affected` if the surviving fork changes are narrower. Keep `Status: active`.
- **Removed**: set `Status` to `removed`. Optionally add a note in `What this changes` explaining when/why the change was removed (e.g., "Removed during upstream sync to v2.4.0 — upstream deleted the file").

After editing, commit:

```bash
git add DOWNSTREAM_CHANGES.md
git commit -m "chore: update DOWNSTREAM_CHANGES.md for upstream $LATEST_TAG sync"
```

### 10. Push and open the MR

```bash
git push --set-upstream origin "$SYNC_BRANCH" --force-with-lease
```

Then open the MR. Prefer `glab`, and capture the MR IID/URL from the result so later steps can post the release-note comment:

```bash
MR_CREATE_OUTPUT="$(glab mr create \
  --source-branch "$SYNC_BRANCH" \
  --target-branch "$DEV_BRANCH" \
  --title "Sync upstream: $LATEST_TAG" \
  --description "$(cat /tmp/mr-body.md)" \
  --remove-source-branch \
  --yes)"
MR_IID="$(printf '%s\n' "$MR_CREATE_OUTPUT" | grep -Eo '![0-9]+' | head -1 | tr -d '!')"
```

If `glab` is unavailable, fall back to the GitLab REST API via `curl` using `$CI_PROJECT_ID` and `$GITLAB_TOKEN` (or `$CI_JOB_TOKEN`). The API path is `POST /projects/:id/merge_requests`.

After the MR exists, fetch upstream release notes for every upstream tag included in this sync (`LAST_TAG..LATEST_TAG`) and add them as a separate MR comment. Prefer `gh` when the upstream URL is GitHub-hosted:

```bash
# For https://github.com/owner/repo.git or git@github.com:owner/repo.git
UPSTREAM_GH_REPO="$(printf '%s\n' "$UPSTREAM_URL" | sed -E 's#^https://github.com/##; s#^git@github.com:##; s#\.git$##')"

RELEASE_NOTES_FILE="/tmp/upstream-release-notes.md"
{
  echo "## Upstream release notes: $LAST_TAG → $LATEST_TAG"
  echo
  for tag in $(git -c versionsort.suffix=-pre tag --merged "refs/tags/$LATEST_TAG" --no-merged "refs/tags/$LAST_TAG" --sort=v:refname 'v[0-9]*' '[0-9]*'); do
    echo "### $tag"
    if gh release view "$tag" -R "$UPSTREAM_GH_REPO" --json body,url --jq 'if .body then .body + "\n\n" + .url else .url end' 2>/dev/null; then
      true
    else
      echo "No GitHub release notes found for this tag."
      echo "https://github.com/$UPSTREAM_GH_REPO/releases/tag/$tag"
    fi
    echo
  done
} > "$RELEASE_NOTES_FILE"
```

If `gh` is unavailable, the upstream URL is not GitHub-hosted, or a tag has no GitHub release, fall back to links: the upstream compare URL when derivable (`https://github.com/<owner>/<repo>/compare/<LAST_TAG>...<LATEST_TAG>`) plus per-tag release/tag URLs when possible. Do not fail the sync only because release notes cannot be fetched; include a clear note in the MR comment.

Add the release-note comment to the MR using `glab` when possible:

```bash
glab mr note "$MR_IID" --message "$(cat "$RELEASE_NOTES_FILE")"
```

If `glab mr note` is unavailable, use the GitLab notes API (`POST /projects/:id/merge_requests/:merge_request_iid/notes`) with `$GITLAB_TOKEN`. Prefer a personal/project token with `api` scope; `CI_JOB_TOKEN` may be insufficient for creating notes depending on the GitLab instance. If the MR already exists from a previous run, update or replace the prior `/upstream-sync` release-notes comment when the tooling supports it; otherwise add a new comment only if the content changed.

**MR description contents** (write to `/tmp/mr-body.md` first):

```
## Upstream sync: <LAST_TAG> → <LATEST_TAG>

Automated sync of upstream tag `<LATEST_TAG>` into `<DEV_BRANCH>`.

### Upstream changelog
Release notes are posted as a separate MR comment after creation. Summary/link: <compare URL or tag/release URL if derivable from the upstream URL>

### Conflicts resolved
<bullet list from step 7's running log; if no conflicts, say "Clean merge, no conflicts.">

### Downstream changes superseded by upstream
<bullet list of `DOWNSTREAM_CHANGES.md` entries replaced by upstream implementations; if none, say "None.">

### DOWNSTREAM_CHANGES.md updates
<if step 9 proposed updates, list what was applied or what is proposed for the reviewer to apply; if no updates needed, say "No ledger updates needed.">

### Reviewer checklist
- [ ] Conflict resolutions preserve fork-specific behavior
- [ ] CI passes
- [ ] `.upstream-version` records `UPSTREAM_REPO=<UPSTREAM_URL>` and `UPSTREAM_VERSION=<LATEST_TAG>`
- [ ] Upstream release notes comment was posted or the MR explains why release notes could not be fetched
- [ ] `DOWNSTREAM_CHANGES.md` entries are accurate — verify any status changes to `superseded` or `removed`

Generated by /upstream-sync.
```

### 11. Cleanup the worktree

After the push and MR creation succeed, remove the worktree to reclaim disk space and avoid leaving stale checkouts:

```bash
cd "$REPO_ROOT"
git worktree remove --force "$WORKTREE_DIR"
```

If the skill exits early (error, already-up-to-date, dry-run), the worktree may still exist. The caller should clean it up:

```bash
# One-liner cleanup (safe even if no worktree exists):
git worktree remove --force "$(git rev-parse --show-toplevel)/.git-upstream-sync-worktree" 2>/dev/null || true
```

In CI, add the cleanup to a `trap` or a post-job step so stale worktrees don't accumulate across runs:

```bash
# At the top of the CI script, before invoking the skill:
cleanup_upstream_sync_worktree() {
    git worktree remove --force "$(git rev-parse --show-toplevel)/.git-upstream-sync-worktree" 2>/dev/null || true
}
trap cleanup_upstream_sync_worktree EXIT
```

### 12. Report

Print a one-line summary to stdout: either `Opened MR !<n>: <title>` or `Already up to date at <tag>`. In CI, this ends up in the job log.

## Dry-run mode

If `UPSTREAM_SYNC_DRY_RUN=1` is set in the environment, do everything through step 9 normally (worktree, branch, merge, conflict resolution, downstream ledger review, tracking-file bump, local commits) but **skip step 10's push and MR creation**. Instead, print the fully-composed MR body to stdout preceded by the line `=== DRY RUN MR BODY ===`, then print the release-note comment body preceded by `=== DRY RUN RELEASE NOTES COMMENT ===`, clean up the worktree (see step 11), and exit 0. This mode exists for local testing and for running the skill against a fixture repo without a real GitLab server — it exercises all the interesting logic without any external side effects.

## Failure modes and what to do

- **Merge fails for a reason other than conflicts** (e.g., upstream history diverged): don't force it. Abort, leave no MR, exit non-zero so the CI job is visibly red. Leave a message explaining what was tried.
- **Conflict that's genuinely unsafe to resolve automatically** (e.g., diverging crypto code, security-sensitive logic): stop, keep the conflict markers in the file, `git add` it anyway, commit with a message flagging this, and say so prominently in the MR description. Humans finish the job. Better to open an MR with flagged conflicts than to silently make a wrong decision.
- **Push rejected**: likely a branch-protection rule on the sync branch namespace. Report and exit non-zero.
- **MR already exists** for this tag: update the existing MR's description rather than opening a duplicate.

## Why the design is this way

- **Tag-based, not branch-based**: tags are stable release points with known quality. Tracking `upstream/main` in a long-lived fork produces a torrent of unreviewable merges. Tags give the human reviewer a discrete, meaningful unit of change.
- **Dotfile for state**: `.upstream-version` lives in the repo so it follows branches and is atomic with the merge commit. Recording both `UPSTREAM_REPO` and `UPSTREAM_VERSION` keeps the sync provenance reviewable and avoids ambiguity when upstream remotes move. No external database, no CI-variable drift, no "which environment is authoritative" question.
- **Always open an MR, never push to dev directly**: even a clean merge deserves a review pass — the reviewer should at least glance at the upstream changelog to spot anything concerning. CI-driven auto-merge would defeat the point.
- **`--force-with-lease`, not `--force`**: if a human has pushed to the sync branch (e.g., fixing a conflict manually), don't stomp on them.
- **Per-tag sync branches**: makes it obvious which MR corresponds to which upstream release, and lets multiple syncs coexist if one gets stuck in review.
- **Git worktree for isolation**: the sync happens in a separate worktree (`.git-upstream-sync-worktree/`) so the main working tree never leaves `DEV_BRANCH`. No stashing, no branch juggling, no risk of a failed sync leaving the repo on the wrong branch. The worktree is cleaned up automatically after the sync completes or can be removed manually with `git worktree remove --force .git-upstream-sync-worktree`.

## Example sessions

**Explicit URL arg (typical CI use):**

```
$ opencode -p "/upstream-sync https://github.com/upstream-org/project.git"

Detected dev branch: main
Using upstream URL from argument; updating existing 'upstream' remote.
Current tracked upstream: v2.3.1
Latest upstream tag: v2.4.0
Creating worktree and branch upstream-sync/v2.4.0...
Merging refs/tags/v2.4.0...
Conflicts in: src/config.rs, package-lock.json
Resolving src/config.rs: upstream added a new field; fork's custom defaults preserved.
Resolving package-lock.json: taking upstream, regenerating via npm install.
Commit: merge + .upstream-version bump
Reviewing DOWNSTREAM_CHANGES.md for superseded entries...
  1 entry superseded by upstream: [feat-telemetry-optout] — upstream added native opt-out in v2.4.0
  User confirmed: updating entry status to superseded.
Commit: update DOWNSTREAM_CHANGES.md
Pushed to origin/upstream-sync/v2.4.0
Opened MR !142: Sync upstream: v2.4.0
Cleaned up worktree .git-upstream-sync-worktree
```

**No arg, reuses existing remote:**

```
$ opencode -p "/upstream-sync"

Detected dev branch: main
Using upstream URL from existing 'upstream' remote: https://github.com/upstream-org/project.git
Already at latest upstream tag: v2.4.0
```
