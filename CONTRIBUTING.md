# Contributing to omadia Proof

`byte5ai/omadia-proof` ist das ANP-Proof-Plugin-Bündel für omadia. Kontext: [plan.md](plan.md) (Implementierungsplan, Codex-reviewed) und [research.md](research.md) (verifizierte Faktenbasis A1–A8). Die Arbeit ist als Issues in Milestones gruppiert (Stufe A + Phase 0–5).

## Dev-Setup

```bash
script/setup   # konfiguriert git-Hooks; installiert Dependencies, sobald Pakete existieren
```

## Workflow

Dieses Repo folgt den byte5ai-Engineering-Standards (`.github/engineering-standards.yml`, Details in [AGENTS.md](AGENTS.md)).

- **Nie im Haupt-Klon committen — immer ein Worktree:**
  ```bash
  git worktree add ../omadia-proof-<feature> -b feat/<desc> main
  ```
  Durchgesetzt durch `.hooks/pre-commit`. Nach Merge aufräumen: `git worktree remove …` + `git branch -D …` (oder `script/prune-worktrees`).
- **Nie direkt auf `main` pushen.** Feature-Branch + PR; CI (`ci`) muss grün sein.
- **Conventional Commits:** `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`, `release:`.
- **Branch-Präfixe:** `feat/ fix/ refactor/ docs/ chore/ test/ release/ dev/`.
- **Keine** `Co-Authored-By:`-Trailer (auch nicht für KI-Agenten). **Nie** `--no-verify`. **Nie** Secrets committen.

## Schema-Backflow (zwingend)

Jede Änderung, die ein ANP-Object-/VC-Schema definiert oder ändert, **MUSS** in derselben Arbeitseinheit einen PR gegen `byte5ai/anp` (Appendix A) eröffnen — siehe [plan.md §1.4](plan.md) und [#2](https://github.com/byte5ai/omadia-proof/issues/2). Der RFC ist kanonisch; bei Divergenz gewinnt die SPEC.
