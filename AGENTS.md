# Agent Instructions

These rules apply to all AI agents working on this repository (Claude, Codex, Copilot, etc.).

## Git Workflow
- **Never push directly to `main`.** All changes go through feature branches and pull requests.
- **Branch naming:** `feat/`, `fix/`, `refactor/`, `docs/`, `chore/`, `test/`, `release/`, `dev/` prefixes.
- **Conventional commits:** `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`, `release:`, `dev:`.
- **No `Co-Authored-By:` trailers for Claude or other AI agents.** Commits are made under the configured git identity, with no model-attribution footer.
- **Never force-push** to any shared branch.
- **Never commit secrets** (`.env`, API keys, tokens, credentials).
- **Never skip hooks** (`--no-verify`).

## Pull Requests
- Keep PR titles short (<70 chars), use conventional prefix.
- One logical change per PR.
- Ensure tests pass before requesting merge.

## Working in a multi-session repo

**Convention (enforced):** the main clone never receives commits. Every change — even a single-line typo fix — lands in a worktree. This is branch-agnostic: an agent whose HEAD got switched to `main` by a parallel session is caught here too, not just one that created a feature branch in the wrong tree. Enforced by the `.hooks/pre-commit` hook shipped via the engineering-standards skill.

~~~bash
git worktree add ../<repo>-<feature> -b <branch> main   # create with new branch
git worktree add ../<repo>-<feature> <existing-branch>  # or attach existing
# work in ../<repo>-<feature>/
git worktree remove ../<repo>-<feature>                 # remove the tree
git branch -D <feature>                                 # remove the branch ref (after merge or discard)
git worktree list                                       # inspect
~~~

**After your PR is merged, clean up your worktree.** The two commands above (`git worktree remove …` + `git branch -D …`) are part of the merge workflow, not optional. For accumulated orphans across the repo, `script/prune-worktrees` (if present) reports and safely removes worktrees whose remote branches are gone or merged.

Build artefacts (`target/`, `node_modules/`, etc.) live per worktree — first build per tree is full cost, subsequent builds are independent.

**Bypass levels** (in increasing persistence):

| Level | Effect | How |
|---|---|---|
| One-off | this commit only | `ALLOW_MAIN_TREE_BRANCH=1 git commit ...` |
| Per repo | persistent disable, other standards still apply | `git config engineering-standards.main-tree-discipline false` |
| Repo exempt | all engineering-standards disabled for this repo | `status: exempt` in `.github/engineering-standards.yml` |

### Drift signals (for any unusual main-tree work that is allowed)

- Run `git branch --show-current` before each commit and confirm it matches what you intended.
- Treat `git status` anomalies as drift signals: directories you didn't touch showing as `??`, unexpected `M` on files you didn't edit. Don't commit through that — `git reflog | head -20` first to see who moved your HEAD.
- Don't reach for `git reset --hard` reflexively; another session may have uncommitted work in the shared tree. Inspect `git stash list` and `git diff --stat` first.

## Pre-push Hook
A `.hooks/pre-push` hook blocks direct pushes to `main`/`master`. Override only when explicitly instructed:
~~~bash
ALLOW_PUSH_TO_MAIN=1 git push origin main
~~~

## Engineering Standards
This repo's engineering-standards status is tracked in `.github/engineering-standards.yml`.
Source of truth: the account's `engineering-standards` repo.
