---
name: pr-reason
description: Use when you have unresolved review comments on a GitHub pull request and want a critical analysis of whether the feedback makes sense before acting on it — categorizes each comment (makes sense / questionable / push back / needs clarification) and flags lessons worth codifying in AGENTS.md. Needs a PR URL or number; owner/repo are derived from the local git remote if not given. Harness-agnostic (plain gh + git, no project-specific dependencies).
---

# PR Reason

Reasons over the *unresolved* review comments on a GitHub PR and tells you which to act on, which to push back on, and which lessons are worth writing into AGENTS.md. Strictly analytical — never edits code or replies on the PR.

## When to use

- A reviewer left feedback and you want a second opinion before implementing it blindly.
- You want to separate "real improvement" from "reviewer is wrong / contradicts our patterns."
- You want to capture recurring review feedback as durable instructions in AGENTS.md.

Not for: writing the PR description or applying the changes (this skill only reasons).

## Inputs

- **Required:** PR URL or number.
- **Owner/repo:** parsed from the URL, else from `gh repo view` / the `origin` remote.

## Workflow

### 1. Resolve target + drift check

```bash
gh pr view <pr> --json number,headRefName,headRefOid,headRepository
git rev-parse HEAD && git branch --show-current
```

Compare the PR's `headRefName`/`headRefOid` to the local branch and HEAD.

- **Match** → local checkout reflects the PR; file-context reasoning is trustworthy.
- **Drift** (different branch, or HEAD behind/ahead) → **say so loudly** and ask whether to continue. Reasoning over line numbers against a drifted tree produces wrong conclusions. Don't silently proceed.

### 2. Fetch unresolved threads

GraphQL — keep only threads where `isResolved == false`:

```bash
gh api graphql -f query='
query($owner:String!,$repo:String!,$pr:Int!){
  repository(owner:$owner,name:$repo){
    pullRequest(number:$pr){
      reviewThreads(first:100){
        nodes{
          isResolved isOutdated path line
          comments(first:20){ nodes{ author{login} body createdAt } }
        }
      }
    }
  }
}' -f owner=<owner> -f repo=<repo> -F pr=<pr>
```

For each unresolved thread keep: latest reviewer comment, `path`, `line`, `isOutdated`.

### 3. Reason over each comment

Read the file around `line` (skip file-context reasoning if drift was detected — note it instead). Judge on three axes:

1. **Makes sense in *this* codebase?** — Does the suggested API/util/symbol actually exist here? Does it fit the module's established patterns? (verify, don't assume)
2. **Good engineering practice generally?** — Sound regardless of this repo: correctness, safety, readability, performance.
3. **Worth an AGENTS.md instruction?** — Only if the lesson is *general and recurring*, not a one-off. A repeated "use guard clauses" → yes. A typo fix → no.

Many comments (5+) → dispatch one subagent per comment for context isolation and run them in parallel. Few → reason inline.

### 4. Report

**Executive summary:** one line on overall feedback quality + drift status.

**Per comment** — `path:line` · category · one-line rationale (cite the code/standard for any "push back"):

| Category | Meaning |
|---|---|
| ✅ Makes sense | Valid and good practice — implement it. |
| 🤔 Questionable | Real tradeoff or partially right — decide case by case. |
| ⛔ Push back | Wrong, or contradicts codebase patterns / best practice — reply, don't apply. Must cite evidence. |
| ❓ Needs clarification | Ambiguous — ask the reviewer before acting. |

**AGENTS.md candidates:** bullet list of generalizable rules worth adding, each with the comment(s) that motivated it and the proposed one-line instruction. If the rule is already in AGENTS.md, say so and skip. Recommend, don't write — the human decides.

## Principles

- Even senior reviewers are wrong sometimes — evaluate objectively.
- Back every "push back" with a code reference or a stated standard. No naked disagreement.
- Verify suggested symbols/APIs exist before calling a comment valid.
- Read-only: no code edits, no PR replies, no AGENTS.md writes.
