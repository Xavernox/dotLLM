---
name: apply-pr-comments
description: Read PR review comments from Gemini/Codex/humans, plan fixes, then apply after approval
disable-model-invocation: true
allowed-tools: Bash, Read, Grep, Glob, Agent, EnterPlanMode, ExitPlanMode
---

# Apply PR Comments Skill

Read review comments on the current branch's PR, analyze them, and plan fixes — entering plan mode so the user approves before any code changes are made.

## Context

Current branch: !`git branch --show-current`
Open PRs for this branch: !`gh pr list --head "$(git branch --show-current)" --json number,title,url --jq '.[] | "#\(.number) \(.title) \(.url)"' 2>/dev/null || echo "(none found)"`

## Instructions

Follow these steps precisely:

### Step 1 — Find the PR

1. Extract the PR from the context above. If multiple PRs exist, use the most recent one.
2. If no PR is found, try: `gh pr view --json number,title,url` (uses current branch).
3. If still no PR, tell the user and stop.

### Step 2 — Fetch all review comments

1. Get PR review comments (inline code comments from reviewers):
   `gh api repos/{owner}/{repo}/pulls/{pr_number}/comments --paginate --jq '.[] | {id, user: .user.login, path: .path, line: .line, body: .body, created_at: .created_at}'`

2. Get PR issue comments (general discussion comments):
   `gh api repos/{owner}/{repo}/issues/{pr_number}/comments --paginate --jq '.[] | {id, user: .user.login, body: .body, created_at: .created_at}'`

3. Get PR reviews (review-level comments with verdict):
   `gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews --paginate --jq '.[] | {id, user: .user.login, state: .state, body: .body}'`

4. Filter out your own comments (from the PR author). Focus on comments from reviewers — typically:
   - `gemini-code-assist[bot]` or similar Gemini bots
   - `openai-codex[bot]` or similar Codex bots
   - `dotllm-claude-code-bot[bot]` (our own bot — skip these)
   - Human reviewers (any other user)

5. If there are NO review comments, tell the user and stop.

### Step 3 — Read referenced files

For each inline comment that references a specific file/line, read that file to understand the context. Use the Read tool to view the relevant code sections.

### Step 4 — Enter plan mode

Use `EnterPlanMode` to enter planning mode. Then:

1. **Summarize each comment** — group by reviewer, show:
   - Reviewer name
   - Comment location (file:line if inline)
   - What they're asking for
   - Your assessment: agree / disagree / needs discussion

2. **For each actionable comment, propose a fix**:
   - What file(s) to change
   - What the change looks like (brief description, not full code)
   - Any concerns or trade-offs

3. **Flag comments to skip** with reasoning:
   - Nitpicks that don't apply to project conventions
   - Suggestions that would hurt performance (critical for dotLLM)
   - Incorrect suggestions (explain why)

4. Present the plan and wait for user approval via `ExitPlanMode`.

### Step 5 — Apply approved fixes

After the user approves (exits plan mode):

1. Implement each approved fix.
2. Tell the user the fixes are applied and **stop here** — do NOT commit, push, or reply to comments.

The user will manually test and benchmark, then use `/create-pr` to commit+push and a separate skill to reply to comments when ready.

### Additional arguments

If `$ARGUMENTS` is provided, use it as additional guidance (e.g., "focus only on Gemini comments", "skip style nits").
