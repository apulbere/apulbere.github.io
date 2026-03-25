---
layout: post
title: Automatic Pull Request comment resolution through AI
tags: [ai, code-review, copilot, claude]
---
Are you tired of Copilot comments on your PR? Then let AI deal with it. In following article you'll find an useful command for Claude Code to automatically resolve pull request comments.

### Prerequisites
* Claude Code
* gh cli. To install it see below instructions:
  * `brew install gh`
  * `gh auth login`

### Create Claude Code command

Create file `~/.claude/commands/resolve-pr-comments.md` with content:

```markdown
etch all open/unresolved review comments on a PR and help me resolve them.

## Step 1: Determine the PR
If a PR number is provided as $ARGUMENTS, use that.
Otherwise, run this command first and capture the output:

gh pr view --json number -q .number

Store the result as the PR number. If the command fails or returns empty, tell me there's no open PR on this branch and stop.

## Step 2: Get Repo Info
Run this command and store the owner and repo name:

gh repo view --json owner,name -q '.owner.login + "/" + .name'

This gives you OWNER/REPO. Use these values in all subsequent commands.

## Step 3: Fetch Unresolved Comments
Use the PR number and OWNER/REPO from previous steps. Replace OWNER, REPO, and PR_NUMBER with the actual values — do NOT use $() substitution.

First, get unresolved review threads via GraphQL:

gh api graphql -f query='query { repository(owner:"OWNER", name:"REPO") { pullRequest(number:PR_NUMBER) { reviewThreads(first:100) { nodes { isResolved, comments(first:10) { nodes { id, body, author { login }, path, line } } } } } } }'

Only include threads where `isResolved` is `false`. Skip any already-resolved threads entirely.

**Important**: Filter out noise before analysis. Ignore any comments from bots like SonarQube/SonarCloud (user login containing "sonar") that indicate passing quality gates, no issues found, or "Passed" status. Only keep Sonar comments that flag actual issues, bugs, or code smells. Also ignore other CI bot comments that are purely status updates (e.g. coverage reports that just show numbers with no actionable feedback).

## Step 4: Critical Analysis
For each comment, provide a structured analysis:

- **Comment**: The reviewer's feedback
- **File & Line**: Where it applies
- **Assessment**: Do you agree this is a valid concern? Be honest — if the comment is nitpicky, subjective, or wrong, say so with reasoning.
- **Recommendation**: One of:
  - **AGREE & FIX** — Valid point, here's the proposed change
  - **DISCUSS** — Has merit but needs discussion, here's a suggested reply
  - **PUSH BACK** — Disagree, here's why and a suggested reply
- **Proposed Change** (if fixing): Show the exact code diff

## Step 5: Present the Plan
Present ALL comments with your analysis in a single summary table, then wait for my approval before making any changes.

## Step 6: Execute (only after approval)
Once I confirm which comments to resolve:
1. Make the code changes for approved fixes
2. For DISCUSS/PUSH BACK items, draft reply comments
3. Stage and show me the final diff before committing
```

### Permissions
This is optional, without it you'll need to approve multiple commands.

Add to `~/.claude/settings.json` the following:
```json
{
  "permissions": {
    "allow": [
      "Bash(gh pr:*)",
      "Bash(gh api:*)"
    ]
  }
}
```
