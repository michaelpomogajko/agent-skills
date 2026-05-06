---
name: pr-review-loop
description: Use when the user asks to commit and push local changes, open a GitHub pull request, or keep automatically fixing PR review comments and failing GitHub checks in the same chat.
---

# PR Review Loop

## Overview

Use this skill to publish the current working tree as a GitHub PR, then create a Codex heartbeat automation that periodically returns to this same thread to address review feedback and failing GitHub checks until no unresolved actionable comments remain and all required GitHub workflows/checks pass.

Prefer composing with `github:yeet` for commit/push/PR creation, `github:gh-address-comments` for thread-aware review-comment handling, and `github:gh-fix-ci` for failing GitHub Actions checks when those skills are available.

## Prerequisites

- A local git repository with a GitHub remote.
- GitHub CLI `gh` installed and authenticated.
- Codex app automation support available through the automation tool.

Run:

```bash
gh --version
gh auth status
git status -sb
```

If `gh` is missing or unauthenticated, stop and tell the user the exact command needed to finish authentication.

## Publish Workflow

1. Inspect scope before staging.
   - Run `git status -sb` and inspect the diff.
   - If changes look mixed or unrelated, ask which files belong in the PR.
   - Stage explicit paths for mixed worktrees; use `git add -A` only when the full working tree is in scope.
2. Choose the branch.
   - If on `main`, `master`, the remote default branch, or a poorly scoped branch, create `codex/<short-kebab-summary>`.
   - If the current branch is already scoped to this work, use it.
3. Run relevant checks before committing when feasible.
   - Use the repo's scripts for typecheck, lint, tests, or build.
   - If checks fail because of the current changes, fix them before publishing.
4. Commit and push.
   - Use a concise commit message that describes the actual change.
   - Push with upstream tracking: `git push -u origin "$(git branch --show-current)"`.
5. Create the PR.
   - Prefer the GitHub app if available; fall back to `gh pr create`.
   - Default to a ready-for-review PR unless the user explicitly asks for a draft.
   - Use the remote default branch as the base unless the user requested another base.

## PR Body Format

Use a clear Markdown body with these sections:

```markdown
## What
- ...

## Why
- ...

## Noteworthy
- ...

## Validation
- ...
```

Keep it factual. Include failed or skipped checks with the reason.

## Automation Setup

After creating the PR, create a heartbeat automation attached to the current thread:

- Kind: `heartbeat`
- Destination: current thread
- Schedule: every 2 minutes
- Status: active
- Name: `PR review loop: <repo>#<number>`

Automation prompt template:

```text
Check <PR_URL> for unresolved actionable review feedback and failing GitHub checks. Use the current repository and branch <BRANCH>. Inspect thread-aware review context and PR check status, including GitHub Actions workflow runs. If there are unresolved actionable PR review comments, requested changes, or failing required checks, implement fixes locally, run relevant local checks, commit the fixes, and push to the same branch. Summarize what was addressed.

If checks are still pending, keep the automation active and report which checks are still running. If there are no actionable comments yet, no review activity has been observed in this loop, and checks are passing or pending, keep the automation active and report that it is waiting for review feedback and check completion.

If review activity has been observed, there are no unresolved actionable comments left, and all required GitHub checks/workflows pass, report that the PR review loop is complete and delete this automation.
```

Use the Codex app automation tool rather than writing raw automation directives by hand. For this app, the recurrence is a 2-minute heartbeat attached to the thread.

When the loop is complete, delete the heartbeat automation with the Codex app automation tool. Do not leave a paused automation behind unless deletion fails; if deletion fails, report the automation id/name and the error.

## Review Loop Rules

- Treat unresolved review threads, requested changes, and inline comments as the source of truth.
- Ignore resolved, outdated, duplicated, informational, or approval-only comments unless they imply a required change.
- If a comment asks for explanation rather than code, draft the response in chat unless the user explicitly asked to post GitHub replies.
- Do not resolve threads, post replies, or submit reviews on GitHub unless the user explicitly asks.
- If feedback conflicts or would cause a regression, stop and explain the tradeoff.
- Each automation iteration that changes code must run relevant checks, commit, and push.
- Keep using the original PR branch; do not open follow-up PRs for review fixes.

## Check Loop Rules

- Treat required PR checks, GitHub Actions workflow conclusions, and `gh pr checks` as the source of truth for CI status.
- Use `github:gh-fix-ci` when available to inspect failing GitHub Actions checks and logs before editing.
- Use `gh pr checks <PR>` or the bundled `gh-fix-ci` inspection script to distinguish failed, pending, skipped, and successful checks.
- For failing GitHub Actions checks, inspect the relevant run/job logs, identify the root cause, implement a focused fix, run the closest local verification, commit, and push.
- For pending checks, wait for a later automation iteration instead of guessing.
- For external non-GitHub Actions checks, report the failing provider and URL. Only attempt a fix when the failure is visible enough to diagnose from available logs or local reproduction.
- If a failing check is unrelated to the PR diff, report that clearly before changing code.
- Do not delete or pause the automation while required checks are failing or pending, unless progress is blocked by authentication, missing secrets, unavailable logs, or an external system outside Codex's reach.

## Completion

Report:

- Branch and PR URL.
- Commit hash or hashes pushed.
- Checks run and results.
- Automation name and cadence.
- Whether the loop is active, waiting for feedback/checks, blocked, or deleted because review feedback is complete and required checks are passing.
