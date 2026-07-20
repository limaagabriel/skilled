---
name: restory
description: Use when a branch has messy, experimental commits and you want to rewrite its history into a small, logical sequence of focused commits before opening a PR — commits that read like the development story and make review easy. Rebuilds the branch from its net diff, dropping add-then-remove noise and right-sizing commit granularity. Rewrites local history only; never pushes. Harness-agnostic (plain git).
---

# Restory

Rewrite the current branch's commits into a clean, review-friendly sequence. The end state (the tree at HEAD) stays **byte-for-byte identical** — only the path there changes: fewer commits, logical order, each one a coherent beat of the story, noise removed.

Use this on a branch full of experimentation you want to tidy before a PR. Not for shared/pushed history you don't own — this rewrites commits.

## When to use

- A feature branch accumulated "wip", "fix", "oops", revert-and-retry commits.
- Code was added in one commit and removed in a later one — pure noise a reviewer shouldn't wade through.
- The commit count is wrong: one giant blob, or dozens of one-line dribbles.

Not for: uncommitted work (that's a commit-planning task), or reordering/rewording commits you want to keep verbatim (that's a plain interactive rebase).

## Core method

The whole diff already exists as `base..HEAD`. The safest rebuild:

1. Reset **HEAD and index** back to the branch base, leaving the working tree untouched.
2. The entire net diff is now unstaged. Add-then-removed code simply isn't there — it equals base, so it never surfaces. Noise gone for free.
3. Carve that net diff into a planned sequence of commits with `git add` (by path, or `-p` by hunk when one file spans commits).
4. Verify the final tree matches the original exactly.

No rebase replay → **no conflicts, ever**. Trade-off: new commits carry the current author/date, and intermediate authorship is lost. For a pre-PR personal cleanup that's fine and usually wanted.

## Workflow

### 1. Establish base + safety

```bash
git branch --show-current
git status --porcelain          # MUST be empty
git symbolic-ref refs/remotes/origin/HEAD --short 2>/dev/null   # default branch
```

- **Working tree not clean** → stop. Ask the user to commit or stash first. Never rewrite over dirty state.
- **On the default branch itself** → stop. This skill operates on feature branches.
- **Base:** `git merge-base HEAD <default>`. If the branch is stacked on another feature branch, ask the user for the correct base — the merge-base with the default branch would swallow the parent branch's commits too.

Create a backup ref before touching anything:

```bash
git branch backup/restory-<branch> HEAD
```

Tell the user the recovery command up front: `git reset --hard backup/restory-<branch>`.

### 2. Analyze the story

```bash
git log --oneline --stat <base>..HEAD     # what the commits currently claim
git diff <base>..HEAD                       # the net truth — this is what gets rebuilt
```

Compare the two. What appears in the log but not the net diff is noise (added-then-removed, reverted experiments, stray debug output) — it drops automatically, name it so the user knows it's intentional.

Design the target commit sequence. Principles:

- **Foundations first.** Types / schema / interfaces / config → implementation → wiring → tests → docs. Each commit should ideally build and pass on its own.
- **One concern per commit.** A reviewer should grasp each in a minute or two.
- **Keep refactors pure.** Don't mix a rename-sweep with a behavior change in the same commit.
- **Separate mechanical/generated from hand-written** (lockfiles, codegen, formatting) so logic commits stay readable.
- **Right-size.** Not one blob, not one-per-file. Typical branch lands around 3–8 commits; scale to actual scope, don't force a number.
- **Messages tell the story.** Imperative subject, the *why* in the body when it isn't obvious. If the existing branch commits use a prefix convention (issue key, branch name), detect and keep it.

### 3. Propose the plan — HARD GATE

Present the proposed sequence and **wait for approval**. Do not rewrite anything before the user approves.

For each planned commit: the message + which files/hunks it takes. Call out explicitly what's being dropped as noise. Example:

```
1. Add config schema for X          config/schema.ts, config/types.ts
2. Implement X resolver             src/resolver.ts
3. Wire resolver into the pipeline  src/pipeline.ts
4. Cover X resolver with tests      test/resolver.test.ts
   (dropped: debug logging added in a1b2c3d and removed in d4e5f6a)
```

### 4. Rebuild

```bash
git reset --mixed <base>        # HEAD+index → base; working tree keeps the full net diff
```

Now the whole net diff is unstaged. For each planned commit, in order:

```bash
git add <paths>                 # or: git add -p <file>   when a file splits across commits
git commit -m "<subject>" -m "<body>"
```

After the last commit `git status` must be clean — nothing left unstaged. If something remains, it belongs in a commit the plan missed; surface it, don't leave it dangling.

### 5. Verify tree integrity — non-negotiable

```bash
git diff backup/restory-<branch> HEAD --stat     # MUST be empty
```

Empty output proves the rewrite preserved the exact end state. **Not empty → the rebuild lost or altered code.** Recover immediately with `git reset --hard backup/restory-<branch>` and report what diverged.

Then show the result:

```bash
git log --oneline <base>..HEAD
```

### 6. Hand back

Report: commits before → after, what was dropped as noise, and the tree-integrity check result. Leave `backup/restory-<branch>` in place — the user deletes it once satisfied. **Never push.**

## Principles

- The final tree is sacred. Verify it (step 5) every time; a "cleaner history" that changed the code is a bug, not a cleanup.
- Approval before rewrite. Gabriel is the sole authority on history changes.
- Rewrite local only. No `git push`, no force-push — even if the branch is already published, surface that and let the human decide.
- Prefer fewer, honest commits over a fake linear story. Don't invent structure the diff doesn't support.
