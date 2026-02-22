---
name: review
description: Review AI-generated PR review comments and discuss improvements
---

## Gather Context

Run the following command to check the current branch:
1. `git branch --show-current` — check current branch

## Identify Target PR

Parse `$ARGUMENTS` to extract a PR number or URL.
If not provided, detect from the current branch:

```
gh pr view --json number --jq '.number' 2>/dev/null
```

If no PR is found, ask the user to specify one.

## Step 1: Collect AI Review Comments

Fetch all review comments on the PR:

```
gh api repos/{owner}/{repo}/pulls/{number}/comments --jq '.[] | {id: .id, path: .path, line: .line, body: .body, user: .user.login, created_at: .created_at}'
```

Also fetch PR-level review summaries:

```
gh api repos/{owner}/{repo}/pulls/{number}/reviews --jq '.[] | {id: .id, user: .user.login, state: .state, body: .body}'
```

Filter to bot/AI-generated reviews (common bot usernames: `github-actions`, `coderabbitai`, `copilot`, or any bot-flagged user).

## Step 2: Analyze Each Suggestion

For each AI review comment:

1. Read the referenced file and surrounding code context (at least 20 lines around the mentioned line).
2. Understand the reviewer's suggestion in the context of the actual code.
3. Evaluate the suggestion's validity:

| Category | Criteria |
|----------|----------|
| **Valid** | Identifies a real bug, security issue, or meaningful improvement |
| **Debatable** | Stylistic preference or trade-off that could go either way |
| **Incorrect** | Misunderstands the code, context, or project conventions |
| **Already handled** | The concern is addressed elsewhere in the codebase |

## Step 3: Present Findings

Group findings by category and present to the user:

```
## Review Analysis for PR #{number}

### ✅ Valid Suggestions (recommended to apply)

1. **[file.py:42]** {summary}
   - Reviewer said: {brief quote}
   - Assessment: {why this is valid, what the fix should be}

### 🤔 Debatable

1. **[file.py:78]** {summary}
   - Reviewer said: {brief quote}
   - Pros: {benefits of applying}
   - Cons: {reasons to skip}

### ❌ Can Safely Ignore

1. **[file.py:15]** {summary}
   - Reviewer said: {brief quote}
   - Why: {reason this is not applicable — e.g., misunderstood context, project convention differs}
```

## Step 4: Discuss and Apply

For each Valid or Debatable suggestion:
1. Ask the user whether to apply the change.
2. If approved, make the code change using Edit.
3. After all approved changes are applied, suggest a commit message following the project's commit convention (e.g., `fix: AI 리뷰 피드백 반영`).

**Important:**
- Do NOT apply any code changes without explicit user approval for each item.
- Read the actual code context before judging — do not rely solely on the review comment.
- Consider the project's existing patterns, conventions, and CLAUDE.md instructions.
- Be honest when an AI review catches a genuine issue — do not dismiss valid feedback.
- If no AI review comments are found, inform the user and suggest running a manual review instead.
