---
name: issue
description: >-
  Create a GitHub issue following project conventions with auto-labeling.
  TRIGGER when: user asks to create an issue, report a bug, request a feature, file a ticket, or track work (e.g., "이슈 만들어줘", "버그 리포트", "feature request", "이거 이슈로 만들어줘").
  DO NOT TRIGGER when: user is viewing, listing, or commenting on existing issues without intent to create a new one.
argument-hint: "[bug|feature|enhancement|chore|docs|...]"
version: "1.3.1"
allowed-tools: Bash(gh issue *), Bash(gh label *), Read, Grep, Glob
---

## Label System

This label scheme is shared with the `/pr` skill for project-wide consistency. The core labels are identical; `/pr` additionally defines `release` for release PRs.

Before assigning a label, ensure it exists in the repository. Color values are 6-character hex without `#`:

```
gh label create "{label}" --color "{hex}" 2>/dev/null || true
```

### Type Labels

| Label | Color | Usage |
|-------|-------|-------|
| `bug` | `d73a4a` | Bug, defect, error |
| `feature` | `0075ca` | New feature request |
| `enhancement` | `a2eeef` | Improvement to existing feature |
| `docs` | `5319e7` | Documentation |
| `chore` | `e4e669` | Maintenance, config, dependencies |
| `refactor` | `d4c5f9` | Code restructuring |
| `style` | `c5def5` | Formatting, linting, whitespace (no code change) |
| `test` | `bfd4f2` | Test-related |
| `perf` | `f9d0c4` | Performance improvement |
| `hotfix` | `b60205` | Urgent production fix |

### Priority Labels (assign when user specifies or urgency is clear)

| Label | Color |
|-------|-------|
| `critical` | `b60205` |
| `high` | `d93f0b` |
| `medium` | `fbca04` |
| `low` | `0e8a16` |

## Determine Issue Type

Parse `$ARGUMENTS` and conversation context:

| Signal | Type |
|--------|------|
| `bug`, error description, "doesn't work", "안 됨", "깨짐", "오류" | Bug |
| `feature`, "추가", "새로운", new capability | Feature |
| `enhance`, improvement to existing | Enhancement |
| `docs`, documentation gap | Docs |
| `chore`, maintenance, config | Chore |
| Ambiguous or unspecified | Ask the user |

## Issue Templates

Write all template content in the language configured in the project's CLAUDE.md.
If no language is configured, follow the user's conversational language.
Examples below are in Korean.

### Bug Report

```markdown
## 설명

{What is happening — clear, specific description of the bug}

## 재현 단계

1. {Step 1}
2. {Step 2}
3. {Step 3}

## 기대 동작

{What should happen}

## 실제 동작

{What actually happens — include error messages or logs if available}

## 환경

- {Runtime, OS, browser, version — only if relevant}

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

### Feature Request

```markdown
## 배경

{Why this feature is needed — problem or opportunity}

## 제안

{What should be built or changed — describe the desired behavior}

## 기대 효과

- {Benefit 1}
- {Benefit 2}

## 참고 사항

- {Related issues, PRs, or external references}
- {Constraints or considerations}

(Omit this section if nothing noteworthy.)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

### General Issue

```markdown
## 설명

{What needs to be done and why}

## 세부 사항

- {Detail 1}
- {Detail 2}

## 참고 사항

- {Related context, links, or references}

(Omit this section if nothing noteworthy.)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Task

1. Determine issue type from `$ARGUMENTS` and conversation context.
2. If the type is ambiguous, ask the user.
3. Gather relevant context from the conversation — code references, error messages, related PRs.
4. Select the matching template and draft the issue title and body.
   - Title: concise, descriptive. No type prefix — the label handles categorization.
   - Fill in template sections from available context. Omit optional sections with no content.
5. Determine labels:
   - **Type label**: always assign based on issue type.
   - **Priority label**: assign only if the user specified priority or urgency is clearly implied.
   - **Project-specific labels**: check the project's CLAUDE.md for custom label conventions.
6. Ensure all labels exist:
   ```
   gh label create "{label}" --color "{hex}" 2>/dev/null || true
   ```
7. Create the issue (use one `--label` flag per label):
   ```
   gh issue create --assignee @me --label "{type_label}" --label "{priority_label}" --title "..." --body "$(cat <<'EOF'
   ...
   EOF
   )"
   ```
   Omit `--label "{priority_label}"` if no priority was assigned. If the command fails due to `--assignee` or `--label` permissions, retry without those flags. Follow the session's tool permission settings for approval.
8. Return the issue URL.

**Important:**
- Do NOT create issues without sufficient context. If the request is too vague, ask for clarification.
- Do NOT assign priority labels unless the user specifies priority or urgency is clearly implied.
- Adapt section headers and content language to the project's CLAUDE.md language setting.
- Always prioritize the project's own issue conventions over the defaults above.
- **Assignee**: Always include `--assignee @me` in `gh issue create`. If it fails due to insufficient permissions, retry without it.
- **Commit references**: Never wrap commit SHAs in backticks (e.g., `` `abc1234` ``). Backtick-wrapped SHAs render as inline code and are not clickable on GitHub. Use plain text (GitHub auto-links SHAs) or explicit markdown links: `[{short_sha}](https://github.com/{owner}/{repo}/commit/{sha})`.
