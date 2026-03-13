---
name: merge-pr
description: Squash-merge the current branch's PR, delete the remote branch, and checkout main
disable-model-invocation: true
allowed-tools: Bash
---

# Merge PR Skill

Squash-merge the current branch's PR into main, delete the remote branch, and switch to an updated main locally.

## Context

Current branch: !`git branch --show-current`
Open PRs for this branch: !`gh pr list --head $(git branch --show-current) --json number,title,url --jq '.[] | "#\(.number) \(.title) \(.url)"' 2>/dev/null || echo "(none)"`
Uncommitted changes: !`git status --short`

## Instructions

### Pre-flight checks

1. Verify the current branch is NOT `main`. If it is, abort with: "Already on main — nothing to merge."
2. Verify there are no uncommitted changes. If there are, abort with: "Uncommitted changes detected — commit or stash before merging."
3. Identify the PR number from the context above. If no open PR exists for this branch, abort with: "No open PR found for this branch."

### Step 1 — Squash-merge the PR

1. Run: `gh pr merge <number> --squash --delete-branch`
   - This squash-merges and deletes the remote branch in one step.
2. If the merge fails (e.g., merge conflicts, failing checks), report the error and stop.

### Step 2 — Switch to main

1. Run: `git checkout main`
2. Run: `git pull` to get the squash-merged commit.

### Step 3 — Clean up local branch

1. The remote branch was already deleted by `--delete-branch`.
2. Delete the local branch: `git branch -d <branch-name>`
   - If `-d` fails (branch not fully merged — shouldn't happen after squash-merge), use `-D` and note it.

### Step 4 — Confirm

1. Run `git log --oneline -3` to show the latest commits on main including the squash-merge.
2. Report: "PR #N merged into main. Local branch `<branch>` cleaned up."
