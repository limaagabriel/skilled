# Codex subagent roles

Codex resolves subagents from standalone TOML files — there is no plugin-manifest
pointer for them (unlike skills). Codex reads agent roles from:

- `.codex/agents/*.toml` — **project-scoped** (these files; active when Codex runs inside this repo)
- `~/.codex/agents/*.toml` — **user-global** (active in every repo)

One file per dispatchable skilled subagent, mirroring the plugin's `agents/*.md`
(Claude Code reads those automatically; Codex needs these TOMLs). Each file binds
only the **model tier + sandbox**; the full role lives in the plugin's
`agents/<name>.md`.

## Install (for running skilled under Codex on other repos)

```sh
mkdir -p ~/.codex/agents
cp /path/to/skilled/.codex/agents/*.toml ~/.codex/agents/
```

(Or symlink them.) When you work *inside* this repo, the project-scoped copies
here are picked up with no install.

## Model tiers

| Role | Claude model | `model` | `model_reasoning_effort` | sandbox |
|---|---|---|---|---|
| code-reviewer | sonnet | `gpt-5.6-terra` | `high` | read-only |

The reviewer role uses `gpt-5.6-terra` with high reasoning for adversarial
scrutiny. Codex fixes a subagent's model in its role file (no dispatch-time
override), so this mapping must live here.
