---
name: init-fork
description: Initialize a soft fork of an upstream project for long-term downstream maintenance. Creates the fork's maintenance infrastructure — DOWNSTREAM_CHANGES.md (the downstream change ledger), .upstream-version (tracking file), the `upstream` git remote, and an AGENTS.md section declaring the repo as a fork — so that the upstream-sync skill can keep the fork in sync with upstream releases going forward. Use this skill whenever the user wants to create a fork, start a soft fork, initialize a fork for maintenance, set up a downstream change ledger, or invokes /init-fork. Designed to pair with /upstream-sync for ongoing maintenance.
---

# Init Fork

Set up a freshly cloned or long-lived GitLab fork for downstream maintenance with a structured change ledger and upstream tracking. Run this once, right after creating or cloning the fork. After that, `/upstream-sync` handles the ongoing sync.

## What this skill does

Given a fork's local clone and an upstream repository URL, the skill:

1. Configures the `upstream` git remote (if not already present).
2. Creates `DOWNSTREAM_CHANGES.md` at the repo root — the structured ledger of all fork-only modifications.
3. Creates `.upstream-version` at the repo root — the tracking file that `/upstream-sync` reads on every run.
4. Appends a **Fork Maintenance** section to `AGENTS.md` (creating the file if needed) that declares the repo as a fork, references the ledger format, and reminds future agents to update `DOWNSTREAM_CHANGES.md` on every downstream change and keep this section at the end of the file.

## Inputs

- **Argument 1 (required)**: upstream repo URL (HTTPS or SSH). This is the project being forked.
- **Current working directory**: a checked-out clone of the fork. `HEAD` is on the development branch.

## Workflow

Run these steps in order.

### 1. Validate environment

- Confirm you're inside a git repo: `git rev-parse --show-toplevel`. `cd` to that root.
- Read the current branch: `git rev-parse --abbrev-ref HEAD`. Save as `DEV_BRANCH`. If detached (`HEAD`), abort — the user needs to check out a real branch first.
- Confirm the working tree is clean (`git status --porcelain` is empty). If dirty, abort — ask the user to commit or stash changes before initializing the fork.

### 2. Set up the upstream remote

```bash
UPSTREAM_URL="$1"  # the argument passed to the skill

if git remote get-url upstream &>/dev/null; then
    EXISTING_URL="$(git remote get-url upstream)"
    if [[ "$EXISTING_URL" != "$UPSTREAM_URL" ]]; then
        echo "Updating 'upstream' remote: $EXISTING_URL -> $UPSTREAM_URL"
        git remote set-url upstream "$UPSTREAM_URL"
    else
        echo "Remote 'upstream' already points to $UPSTREAM_URL"
    fi
else
    git remote add upstream "$UPSTREAM_URL"
    echo "Added 'upstream' remote: $UPSTREAM_URL"
fi

git fetch upstream --tags
```

If `UPSTREAM_URL` is empty, abort with: "No upstream URL provided. Usage: /init-fork <upstream-url>"

### 3. Discover the baseline upstream tag

The fork was created from upstream at some point. Find which upstream tag the fork's history contains, so `.upstream-version` starts with an accurate baseline rather than blindly recording "latest." This mirrors the bootstrap logic in `/upstream-sync`:

```bash
LATEST_TAG=$(git -c versionsort.suffix=-pre \
  for-each-ref --sort=-v:refname --format='%(refname:short)' \
  'refs/tags/v[0-9]*' 'refs/tags/[0-9]*' \
  --count=1)

BASELINE_TAG=""
while read -r tag; do
    [[ -z "$tag" ]] && continue
    if git merge-base --is-ancestor "refs/tags/$tag" HEAD 2>/dev/null; then
        BASELINE_TAG="$tag"
        break
    fi
done < <(git -c versionsort.suffix=-pre \
    for-each-ref --sort=-v:refname --format='%(refname:short)' \
    'refs/tags/v[0-9]*' 'refs/tags/[0-9]*')

if [[ -z "$BASELINE_TAG" ]]; then
    echo "Warning: no upstream tag found in fork ancestry. Using latest tag ($LATEST_TAG) as baseline."
    echo "If this fork shares history with upstream, the first /upstream-sync run will discover the correct baseline."
    BASELINE_TAG="$LATEST_TAG"
fi
```

### 4. Create `.upstream-version`

```bash
cat > .upstream-version <<EOF
UPSTREAM_REPO=$UPSTREAM_URL
UPSTREAM_VERSION=$BASELINE_TAG
EOF
```

If `.upstream-version` already exists, ask the user whether to overwrite it. Show the current contents and explain that `/upstream-sync` relies on this file.

### 5. Create `DOWNSTREAM_CHANGES.md`

If `DOWNSTREAM_CHANGES.md` already exists, do not overwrite it — just verify it follows the expected format and tell the user it's already present. If it doesn't exist, create it with the standard header:

```markdown
# Downstream Changes

This file is the source of truth for all intentional fork-only modifications.
The upstream-sync skill reads it during merge conflict resolution to preserve
fork behavior. Update this file every time you add, modify, or remove a
fork-only change.

---

<!-- Add new entries below using the format described in AGENTS.md. -->
```

The file starts with only the header. The next step may pre-populate entries from existing fork-only commits.

### 5b. Detect and populate existing downstream changes

The fork may already contain commits that don't exist upstream. This step discovers those commits, presents them to the user, and offers to auto-generate `DOWNSTREAM_CHANGES.md` entries from their content. This is especially valuable for forks that have been maintained informally before adopting this workflow.

#### Find fork-only commits

Using the `BASELINE_TAG` discovered in step 3, list commits on the dev branch that are not reachable from the upstream tag:

```bash
FORK_COMMITS=$(git log --format='%H %s' refs/tags/"$BASELINE_TAG"..HEAD --no-merges)
```

If `BASELINE_TAG` is empty (no shared history), fall back to listing all commits and note that everything appears fork-only — this is likely a re-initialized repo and the user should cherry-pick carefully.

#### Present the commits

If there are fork-only commits, display them to the user in a readable format:

```
Found N fork-only commits (not present upstream at BASELINE_TAG):

  abc1234  Add custom telemetry opt-out
  def5678  Patch config defaults for internal deployment
  9012345  Remove upstream analytics tracking
  ...

Would you like to populate DOWNSTREAM_CHANGES.md from these commits?
You can select all, none, or specific commits to document.
```

Present the list and ask the user to choose. The user may say:
- **All** — inspect every commit and generate entries for all of them
- **None** — leave `DOWNSTREAM_CHANGES.md` empty, proceed to step 6
- **A selection** — only inspect specific commits (by short hash or number)

#### Inspect selected commits

For each commit the user selects, inspect its content to generate a `DOWNSTREAM_CHANGES.md` entry:

1. Read the full commit diff: `git show --stat <sha>` for the file list, `git show <sha>` for the full diff.
2. Classify the change by reading both the commit message and the actual diff content:
   - Does it add new files or functions? → `feature`
   - Does it fix a bug or adjust logic? → `patch`
   - Does it change configuration, defaults, or flags? → `config`
   - Does it replace upstream behavior with fork-specific behavior? → `override`
   - Does it remove or disable upstream code? → `removal`
3. Generate a slug ID from the commit's purpose (e.g., `feat-telemetry-optout`, `patch-config-defaults`, `rm-analytics-tracking`).
4. Determine scope from the file paths in the diff.
5. Write the entry in the standard format.

Group closely related commits into a single entry where it makes sense — if three consecutive commits all iterate on the same feature, they become one entry with all three SHAs in the `Introduced` field and a consolidated description. Use your judgment: the goal is one entry per logical downstream change, not one entry per commit.

#### Write entries to DOWNSTREAM_CHANGES.md

Append the generated entries after the header, before the HTML comment. Each entry follows the format documented in the AGENTS.md template:

```markdown
## [slug-id]: Short description

- **Scope**: `path/to/file`
- **Type**: patch | feature | config | override | removal
- **Status**: active
- **Introduced**: <sha> (or multiple SHAs for grouped entries)
- **Superseded by upstream**: N/A

### What this changes

<description inferred from the commit message and diff>

### Files affected

- `path/to/file`: <summary of what was changed>
```

If no fork-only commits are found, or the user declines, skip this step — the ledger remains empty and ready for future entries.

### 6. Append the Fork Maintenance section to `AGENTS.md`

Read `AGENTS.md` if it exists. Check whether a `## Fork Maintenance` section already exists. If it does, verify it matches the expected format and skip. If it doesn't, append the following section **at the very end of the file**.

If `AGENTS.md` doesn't exist, create it with just this section.

The section to append:

```markdown

---

## Fork Maintenance

This repository is a **soft fork** of an upstream project. It tracks upstream
tagged releases and carries a small number of intentional downstream changes.

### Maintenance files

| File | Purpose |
|------|---------|
| `.upstream-version` | Tracks the upstream repo URL and the last merged upstream tag. Written by `/upstream-sync` on every sync. |
| `DOWNSTREAM_CHANGES.md` | Ledger of all fork-only modifications. Read by `/upstream-sync` during conflict resolution to preserve downstream behavior. |

### DOWNSTREAM_CHANGES.md format

Every entry in `DOWNSTREAM_CHANGES.md` follows this structure:

```markdown
## [slug-id]: Short description

- **Scope**: `path/to/file` (or comma-separated paths)
- **Type**: patch \| feature \| config \| override \| removal
- **Status**: active \| superseded \| removed
- **Introduced**: <commit-sha, tag, or date>
- **Superseded by upstream**: <upstream-version or N/A>

### What this changes

Plain-English description of what the fork does differently from upstream and why.

### Files affected

- `path/to/file`: what was changed (function names, config keys, line ranges)
```

**Type values:**
- `patch` — bug fix applied downstream ahead of upstream
- `feature` — new functionality not present upstream
- `config` — configuration changes, default values, feature flags
- `override` — behavior replacement (fork's implementation supersedes upstream's)
- `removal` — upstream code intentionally removed or disabled in the fork

**Status values:**
- `active` — currently in effect
- `superseded` — upstream now implements equivalent functionality; entry kept for history
- `removed` — change reverted; entry kept for history

### When making downstream changes

Every time you make a fork-only modification — adding a feature, patching a bug,
changing a default, overriding behavior — you **must** add or update an entry in
`DOWNSTREAM_CHANGES.md`. This is not optional. Without it, `/upstream-sync` has
no way to know which changes to preserve during upstream merges, and downstream
modifications will be silently overwritten.

When `/upstream-sync` detects that an upstream release implements the same
feature or fix as a downstream change, it will update the entry's status to
`superseded` and note the upstream version.

<!-- IMPORTANT: This section must remain at the end of AGENTS.md. Do not move it or add content after it. -->
```

Note the HTML comment at the end — this is the guard rail that tells future agents not to move or bury this section.

### 7. Commit the fork infrastructure

```bash
git add .upstream-version DOWNSTREAM_CHANGES.md AGENTS.md
git commit -m "chore: initialize fork maintenance infrastructure

- Add .upstream-version tracking upstream at $BASELINE_TAG
- Add DOWNSTREAM_CHANGES.md ledger (<N> entries pre-populated from fork history / empty, ready for entries)
- Add Fork Maintenance section to AGENTS.md

Paired with /upstream-sync for ongoing upstream release tracking."
```

If any of these files were already present and unchanged, skip them in the `git add`. Adjust the commit message based on whether entries were pre-populated ("3 entries pre-populated from fork history") or left empty ("empty, ready for entries").

### 8. Report

Print a summary:

```
Fork initialized for downstream maintenance.

  Upstream remote:  <UPSTREAM_URL>
  Baseline tag:     <BASELINE_TAG>
  Dev branch:       <DEV_BRANCH>

Created:
  .upstream-version       — tracking file (upstream-sync reads this)
  DOWNSTREAM_CHANGES.md   — change ledger (<N> entries from existing fork history / empty)
  AGENTS.md (updated)     — Fork Maintenance section appended

Next steps:
  1. Review the pre-populated entries in DOWNSTREAM_CHANGES.md (if any) for accuracy
  2. Make additional downstream changes and document each one in DOWNSTREAM_CHANGES.md
  3. Run /upstream-sync periodically to pull upstream releases
```

## Edge cases

- **AGENTS.md has existing content**: The section is appended at the end. The skill checks for an existing `## Fork Maintenance` heading to avoid duplication.
- **Repo is not yet pushed to GitLab**: The skill only touches local files and the git remote config. Pushing is the user's responsibility.
- **`.upstream-version` already exists**: Ask before overwriting — the user may be re-initializing after a partial setup.
- **No upstream tags found**: Still create the infrastructure. `.upstream-version` records `UPSTREAM_VERSION=none` and the first `/upstream-sync` run will handle the bootstrap when tags appear.
- **Fork has no shared history with upstream** (re-initialized repo): The skill warns prominently but proceeds. The first `/upstream-sync` run will detect this and handle it.
- **Many fork-only commits**: If there are more than ~30 fork-only commits, warn the user that the list is large and suggest they review carefully. Group related commits aggressively to keep the ledger manageable. The goal is a concise change ledger, not a 1:1 commit mirror.
- **Merge commits in fork history**: Exclude merge commits from the fork-only list (`--no-merges`). Only non-merge commits represent intentional downstream changes. If a merge commit was used to pull in another branch's fork changes, those changes are already captured by the non-merge commits on that branch.
