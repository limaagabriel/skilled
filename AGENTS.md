# AGENTS.md

## What this is

A collection of harness-agnostic agent **skills**. One repo, consumed by multiple coding agents.

Each skill lives in `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`) followed by the instructions. The frontmatter format is shared — both target harnesses read the same file.

## Target harnesses

- **Claude Code** — discovers skills via `.claude-plugin/plugin.json`; auto-loads every `skills/*/SKILL.md`.
- **Codex CLI** — discovers skills via the `skills` field in `.codex-plugin/plugin.json` (points at `./skills`).
- **pi coding agent** — discovers skills via the `pi.skills` field in `package.json` (points at `./skills`).

All three read the same `SKILL.md` files. Keep skills harness-agnostic: plain tools (`gh`, `git`, shell), no project- or harness-specific dependencies.

Keep the version string in lockstep across `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json`, and `package.json`.

## Running under Codex CLI

Skills are written in Claude Code's vocabulary (the Agent tool, `subagent_type`, model names like `sonnet`). Under Codex, read every such mention through these equivalences — the intent is identical, only the mechanism differs:

- **"dispatch X via the Agent tool / `subagent_type: X`"** → invoke the Codex agent role **X** from `.codex/agents/X.toml` (install per `.codex/agents/README.md`). Codex reads agent roles from those standalone TOMLs, not from the plugin manifest.
- **A subagent's `model:`** (`sonnet`, etc.) → the role's `.codex/agents/X.toml` (`model` + `model_reasoning_effort`); ignore the Claude model name.
- **Fresh/stateless subagent** → spawn a new Codex role each time; never resume a prior one.

## Adding a skill

1. Create `skills/<name>/SKILL.md`.
2. Add frontmatter: `name` (kebab-case, matches dir) and `description` (when to use it).
3. Write the instructions below the frontmatter.

No manifest edits needed — all harnesses auto-discover the `skills/` directory. If the skill dispatches a subagent, add a matching `.codex/agents/<name>.toml` so Codex can resolve the role.
