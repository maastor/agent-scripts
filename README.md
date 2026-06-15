# Agent Scripts

Shared, reusable skills for my local agent workspaces (Claude Code / Codex-style agents).

This repo is the canonical place for:

- `skills/`: reusable workflow skills, discovered automatically by agents.

> Shared agent rules (`AGENTS.md`), portable helper `scripts/`, and `release/` tooling
> are intentionally not part of this repo yet.

## Skills

Skills are the main routing layer. Each `skills/<name>/SKILL.md` has YAML front matter:

```yaml
---
name: skill-name
description: "Short generic trigger phrase."
---
```

Rules:

- Keep descriptions short and generic; optimize for routing, not documentation.
- Keep skill bodies terse and operational.
- Prefer helper scripts under `skills/<name>/scripts/` when a workflow has repeatable commands.
- Put longer docs under `skills/<name>/references/` and load them on demand.
- Quote `description` in front matter.

### Layout

```text
skills/
  <name>/
    SKILL.md            # required: front matter + operational body
    scripts/            # optional: repeatable commands
    references/         # optional: long-form docs, loaded on demand
    agents/             # optional: agent definitions (e.g. openai.yaml)
```

## Discovery

Point global skill discovery here so every workspace sees the same skills:

```text
~/.claude/skills  -> ~/Documents/projects/agent-scripts/skills
~/.codex/skills   -> ~/Documents/projects/agent-scripts/skills
```

Shared personal skills live as real folders in `skills/`. Skills owned by another repo
can be exposed here with a tracked relative symlink, for example:

```text
skills/autoreview -> ../../agent-skills/skills/autoreview
```
