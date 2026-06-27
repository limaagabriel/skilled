# AGENTS.md

## What this is

A collection of harness-agnostic agent **skills**. One repo, consumed by multiple coding agents.

Each skill lives in `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`) followed by the instructions. The frontmatter format is shared — both target harnesses read the same file.

## Target harnesses

- **Claude Code** — discovers skills via `.claude-plugin/plugin.json`; auto-loads every `skills/*/SKILL.md`.
- **pi coding agent** — discovers skills via the `pi.skills` field in `package.json` (points at `./skills`).

Both read the same `SKILL.md` files. Keep skills harness-agnostic: plain tools (`gh`, `git`, shell), no project- or harness-specific dependencies.

## Adding a skill

1. Create `skills/<name>/SKILL.md`.
2. Add frontmatter: `name` (kebab-case, matches dir) and `description` (when to use it).
3. Write the instructions below the frontmatter.

No manifest edits needed — both harnesses auto-discover the `skills/` directory.
