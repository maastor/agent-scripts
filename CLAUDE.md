# CLAUDE.md — agent-scripts

Personal repo of reusable agent skills. See `README.md` for layout and the
`skills/<name>/SKILL.md` convention.

## Branching

- No direct pushes to `main` (blocked by a global pre-push hook). Use feature branches + PRs.

## Local git identity

This repo commits and pushes as `maxi-scala`, not the global identity. The machine-local
identity and SSH-auth setup is documented in `.github/CLAUDE.md`, which is intentionally
untracked (see `.gitignore`).
