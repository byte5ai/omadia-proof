# Contributing to omadia Proof

`byte5ai/omadia-proof` is the ANP-Proof plugin bundle for omadia. Context: [plan.md](plan.md) (the Codex-reviewed implementation plan) and [research.md](research.md) (the verified fact base, A1–A8). Work is tracked as issues grouped into milestones (Stage A + Phase 0–5).

## Dev setup

```bash
script/setup   # configures git hooks; installs dependencies once packages exist
```

## Workflow

This repo follows the byte5ai engineering standards (`.github/engineering-standards.yml`, details in [AGENTS.md](AGENTS.md)).

- **Never commit in the main clone — use a worktree:**
  ```bash
  git worktree add ../omadia-proof-<feature> -b feat/<desc> main
  ```
  Enforced by `.hooks/pre-commit`. After merge, clean up: `git worktree remove …` + `git branch -D …` (or `script/prune-worktrees`).
- **Never push directly to `main`.** Feature branch + PR; CI (`ci`) must be green.
- **Conventional commits:** `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`, `release:`.
- **Branch prefixes:** `feat/ fix/ refactor/ docs/ chore/ test/ release/ dev/`.
- **No** `Co-Authored-By:` trailers (including for AI agents). **Never** `--no-verify`. **Never** commit secrets.

## Schema backflow (mandatory)

Any change that defines or alters an ANP object/VC schema **MUST** open a matching PR against `byte5ai/anp` (Appendix A) in the same unit of work — see [plan.md §1.4](plan.md) and [#2](https://github.com/byte5ai/omadia-proof/issues/2). The RFC is canonical; on divergence, the SPEC wins.
