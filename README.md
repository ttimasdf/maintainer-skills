# Maintainer Skills

[中文](README.zh-CN.md)

Opencode skills for maintaining long-lived GitLab forks of upstream projects. Two skills that work together: **init-fork** sets up the fork infrastructure, **upstream-sync** keeps it in sync with upstream releases.

## Skills

### `/upstream-sync`

Sync a GitLab fork with the latest tagged release from an upstream repository.

- Fetches the newest semver tag from upstream
- Merges into the dev branch via an isolated git worktree
- Resolves merge conflicts intelligently, using `DOWNSTREAM_CHANGES.md` as the source of truth for fork-specific behavior
- Detects when upstream supersedes a downstream change and proposes ledger updates
- Opens a GitLab merge request with upstream release notes
- Tracks state in `.upstream-version` — no external database

Designed to run unattended in CI (`opencode -p "/upstream-sync [URL]"`) or interactively.

### `/init-fork`

Initialize a soft fork for long-term downstream maintenance. Run once after cloning the fork.

- Configures the `upstream` git remote
- Creates `.upstream-version` with the discovered baseline tag
- Creates `DOWNSTREAM_CHANGES.md` — the structured ledger of fork-only modifications
- Appends a Fork Maintenance section to `AGENTS.md`
- Detects existing fork-only commits and offers to pre-populate the ledger

Pairs with `/upstream-sync` for ongoing maintenance.

## Quick Start

```bash
# 1. Clone the fork
git clone https://gitlab.example.com/your-group/your-fork.git
cd your-fork

# 2. Initialize the fork (one-time)
opencode -p "/init-fork https://github.com/upstream/project.git"

# 3. Sync with upstream (run periodically or in CI)
opencode -p "/upstream-sync"
```

## File Structure

```
skills/
  upstream-sync/SKILL.md    # Sync skill definition and workflow
  upstream-sync/evals/       # Evaluation test cases
  init-fork/SKILL.md         # Init skill definition and workflow
```

## Maintained Artifacts

When these skills run on a fork, they create and manage:

| File | Purpose |
|------|---------|
| `.upstream-version` | Tracks the upstream repo URL and last merged tag |
| `DOWNSTREAM_CHANGES.md` | Ledger of all fork-only modifications |
| `AGENTS.md` (Fork Maintenance section) | Declares the repo as a fork and documents the ledger format |

## License

Internal use.
