---
name: code-review
description: Multi-perspective code review. Default - reviews LOCAL current-branch changes. Pass a GitHub PR URL or PR# to review a REMOTE PR instead. Read-only, report-only - returns findings plus a verdict, never applies fixes or posts comments. Triggers - "review my branch", "code review this", "pre-PR review", "review PR <url/number>".
---

# Code Review

Coordinates an independent, isolated review of a diff and reports findings plus a verdict. This file is the orchestration layer; the reviewing doctrine itself lives in `references/reviewer.md` and `references/specialists/*.md`.

## When to use / not

- Reviewing a working branch's changes before opening a PR.
- Reviewing an already-open PR before merge.
- NOT for posting comments back to GitHub/GitLab, applying fixes, or process/workflow advice (review-speed SLOs, reviewer-roulette, approval workflows). This skill only reports.

## Inputs

- No argument → review the local branch's changes.
- A PR URL or number → review that PR instead.

## Workflow

### 1. Resolve target + gates

**Not a git repo:** if `git rev-parse --is-inside-work-tree` fails, abort loudly — say plainly this isn't a git repository and there's nothing to review.

**Local (no arg):**

```bash
git rev-parse --is-inside-work-tree
git rev-parse --abbrev-ref origin/HEAD 2>/dev/null || echo main
git merge-base <default-branch> HEAD
git status --short                 # staged + unstaged + untracked
git add -N .                       # intent-to-add: pulls untracked files' content into the diff below, stages nothing for real
git diff <merge-base>               # committed branch commits + staged + unstaged, through the working tree
```

The diff to review is committed branch commits **through the working tree** — merge-base..HEAD, plus whatever's staged and unstaged on top. Treat all of it as one change: a reviewer cares about what would ship if you committed and opened the PR right now, not about what happens to already be committed. `git diff <merge-base>` alone never shows the content of untracked (`??`) files — a brand-new file that was never `git add`ed would otherwise ship unreviewed. `git add -N` (intent-to-add) fixes this: it registers each untracked path so its content appears in the diff, without actually staging or committing anything. If `origin/HEAD` isn't set, fall back to `main` or `master` (whichever exists).

Also collect the commit messages between merge-base and HEAD — this is the change description for local review (see step 2).

**PR arg (URL or number):**

```bash
gh pr view <pr> --json title,body,baseRefName,headRefName
gh pr diff <pr>
```

Parse owner/repo from the URL if given; otherwise resolve via `gh repo view`. The PR title + body is the change description.

**Empty diff:** if there's nothing to review (no commits ahead and no working-tree changes, or an empty PR diff) — abort loudly, say so, don't fabricate a review.

### 2. Navigate order (Google)

The reviewer itself owns the read order (description first, then the main part of the change, then the rest — see `references/reviewer.md`). The coordinator's job here is just to make sure the reviewer *can* follow that order: always pass the change description (commit messages or PR title/body) **alongside** the diff, never the diff alone.

### 3. Trivial fast path (GitLab)

If the change is unambiguously trivial — a typo/copy fix, a tiny no-behavior-change refactor, removing dead code, or a well-understood logic change under ~5 lines — dispatch through the same isolated mechanism as step 4, just a single lightweight pass with no specialist fan-out (skip step 5 entirely). Otherwise proceed to full dispatch (step 4) followed by specialists (step 5). Either way, the reviewer always runs in its own isolated dispatch — the trivial path is a lighter pass, never an excuse to review inline.

### 4. Dispatch the reviewer in isolation

The single source of truth for reviewing doctrine is `references/reviewer.md`, resolved relative to this skill's own directory. **Portability rule: read that file's content yourself and inject it into the dispatched reviewer's prompt — never tell the subagent to open the path itself.** This skill may be installed anywhere, and the dispatched process's cwd is the user's project, not this repo, so a relative path won't resolve for it.

- **Claude Code (has subagents):** dispatch the Agent tool with `subagent_type: code-reviewer`, prompt = full contents of `references/reviewer.md` + the change description + the diff (+ any triggered specialist file contents from step 5, each clearly labeled).
- **pi / a harness without custom subagents:** spawn a fresh read-only process: `pi -p --exclude-tools write,edit "<reviewer.md contents + description + diff>"`. The `--exclude-tools write,edit` flag is what enforces read-only.
- **Never review inline in the coordinator's own context.** A reviewer that shares the builder's context isn't independent — that independence is the entire point of this skill.

### 5. Conditional specialists

Specialists run strictly **after** the primary pass completes, never alongside it — a specialist needs the primary reviewer's actual findings for context (see below), which don't exist until that pass finishes. Once the primary pass is in, inspect the diff for these trigger domains — by diff content only, never by asking the primary reviewer to decide:

- Touches DB migrations, schema, raw SQL, or expensive-query-shaped patterns → also dispatch the `database` specialist.
- Touches credentials/tokens/secrets, authn/authz, trust-boundary input handling, or new/changed dependencies (lockfiles) → also dispatch the `security` specialist.
- Touches user-facing templates, DOM, CSS, or UI components → also dispatch the `frontend-ux` specialist.

Each specialist pass is its own isolated dispatch (same mechanism as step 4), with prompt = `references/reviewer.md` + that specialist's `references/specialists/<name>.md` + the change description + the diff + the primary reviewer's findings. A trivial-fast-path change (step 3) runs no specialists, regardless of what it touches.

### 6. Merge + report

Collect the primary pass and every specialist pass. Dedup overlapping findings (same `file:line` raised by more than one pass), rank remaining findings by severity with `Blocking` first. Emit one merged report in the exact output-contract shape from `references/reviewer.md`:

1. **Summary** — one line, overall code-health impact.
2. **Findings** — `file:line`, verbatim quote, the why, exactly one severity label.
3. **Praise** — what's done well and why.
4. **Verdict.**

Recompute the final Verdict yourself across all passes — don't just take the primary reviewer's verdict at face value:

- `REQUEST CHANGES` if **any** pass (primary or specialist) produced a `Blocking` finding.
- else `APPROVE WITH NITS` if only `Nit:`/`Optional:`/`FYI:` findings remain anywhere.
- else `APPROVE`.

## Out of scope

Posting comments to GitHub/GitLab, applying fixes, review-speed/SLO rules, reviewer-roulette or approval-workflow policy. This skill is report-only and read-only, end to end.
