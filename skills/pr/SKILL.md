---
name: pr
description: |
  Create a pull request following project conventions.
  TRIGGER when: user asks to create/open a PR, push and create PR, or any workflow that includes PR creation (e.g., "PR 올려줘", "푸시하고 PR 만들어줘", "commit, push, and create PR").
  DO NOT TRIGGER when: user is viewing or listing existing PRs, or performing git operations without PR intent.
version: "1.1.4"
---

## Determine Base Branch

Resolve the target base branch before proceeding. Use the first matching rule:

1. **Explicit override**: If `$ARGUMENTS` contains `--base <branch>`, use that branch.
2. **Project config**: Read the project's CLAUDE.md for a `Branch Strategy` section:
   ```
   ## Branch Strategy
   - default-base: dev
   - release-base: main
   - direct-to-release: hotfix/*
   ```
   Apply rules:
   - Release PR → use `release-base`
   - Current branch matches a `direct-to-release` pattern → use `release-base`
   - Current branch equals `default-base` (e.g., you are on `dev`) → use `release-base`
   - Otherwise → use `default-base`
3. **Auto-detect** (if no project config found):
   a. Check for an integration branch: `git branch -r | grep -E 'origin/(dev|develop)$'`
   b. If found:
      - Current branch IS the integration branch (`dev`/`develop`) → base = `main`
      - Otherwise, compare fork points:
        ```
        dev_base=$(git merge-base HEAD origin/dev 2>/dev/null)
        main_base=$(git merge-base HEAD origin/main 2>/dev/null)
        ```
        The more recent merge-base indicates the parent branch. If equal, default to the integration branch.
   c. If no integration branch exists → use the repo default branch.

Do NOT ask for user confirmation. Instead, briefly state which base branch was selected and why (one line) when presenting the PR.

## Gather Context

Run the following commands using the resolved `{base-branch}`:
1. `git log --oneline {base-branch}..HEAD 2>/dev/null || git log --oneline -10` — list commits on this branch
2. `git diff {base-branch}...HEAD --stat 2>/dev/null || echo "Could not determine diff"` — list changed files

## Determine PR Type

Parse `$ARGUMENTS`:
- If it contains **"release"** → use the **Release PR Template** below.
- Otherwise → use the **Individual PR Template**.

---

## PR Title Convention

Format: `{Type}: {description}`

### Type Prefixes (capitalize first letter — differs from commit messages)

| Type | Purpose |
|------|---------|
| `Feat:` | New feature |
| `Fix:` | Bug fix |
| `Refactor:` | Code restructuring |
| `Perf:` | Performance improvement |
| `Docs:` | Documentation |
| `Test:` | Test changes |
| `Chore:` | Maintenance |
| `Hotfix:` | Urgent fix |
| `Release:` | Release integration (e.g., `Release: {default-base} → {release-base} 통합 (v0.5.0)`) |

Write the description in the language configured in the project's CLAUDE.md.
If no language is configured, follow the user's conversational language.

---

## Individual PR Template

Apply this template for feature, fix, refactor, and other non-release PRs.

```markdown
## 개요

{Background → motivation/problem → approach, in 2-4 sentences. Focus on WHY this change was needed and WHAT approach was taken.}

> ⚠️ **Breaking Change**: {only if applicable — describe migration needed}

## 변경 사항

### 1. {Change category title}

{Detailed description. Include before/after comparisons, code snippets, or diagrams where helpful.}

### 2. {Change category title}

{...}

## 참고 사항

- {Points reviewers should focus on}
- {Intentional omissions and reasons}
- {Follow-up work if any}

(Omit this section entirely if nothing noteworthy.)

## 관련 이슈/PR

- #{number}

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

---

## Release PR Template

Apply this template when `$ARGUMENTS` contains "release". Typically used for `{default-base} → {release-base}` integration.

Collect all merged PRs since the last release:
```
gh pr list --state merged --base {default-base} --limit 30 --json number,title --jq '.[] | "- #\(.number) \(.title)"'
```

Title format: `Release: {default-base} → {release-base} 통합 (vX.Y.Z)`

```markdown
## 개요

{Release summary — what this release includes}

> ⚠️ **배포 공지**: {Deployment impact notice — migration steps if any}

## 주요 변경사항

### 1. {Feature/Fix name} (#{PR number})

{Brief summary of this PR's changes}

### 2. {Feature/Fix name} (#{PR number})

{...}

## 참고 사항

- {Deployment considerations}
- {Follow-up work if any}

(Omit this section entirely if nothing noteworthy.)

## 관련 PR

- #{number1}
- #{number2}

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

---

## Task

1. **Resolve base branch** following the "Determine Base Branch" rules above.
2. Determine PR type: Release (if `$ARGUMENTS` contains "release") or Individual.
3. Sync check before pushing:
   - `git fetch origin {base-branch}`
   - If the local branch is behind, inform the user and suggest an appropriate action (rebase, merge, or proceed as-is).

### If Individual PR:

4. Push the branch if not already pushed: `git push -u origin {branch-name}`
5. Draft the PR title and body using the Individual PR Template, and create the PR: `gh pr create --base {base-branch} --assignee @me --title "..." --body "$(cat <<'EOF' ... EOF)"`. Follow the session's tool permission settings for approval.
6. Return the PR URL. Include a one-line note: **Base branch: `{base-branch}`** — {reason} (e.g., "CLAUDE.md Branch Strategy 설정에 따라 결정" or "origin/dev에서 분기한 브랜치로 감지").

### If Release PR:

4. **Determine version:**
   a. Check `$ARGUMENTS` for an explicit version (e.g., `/pr release v1.2.0`).
   b. If not provided, detect the current version from the project:
      - Read the project's CLAUDE.md for release process instructions or version conventions.
      - Search for version sources: `package.json`, `pyproject.toml`, `Cargo.toml`, `marketplace.json`, or similar files that already exist in the project.
      - Check recent git tags: `git tag --sort=-v:refname | head -5`
   c. Analyze included changes (via merged PR titles or commit messages) and recommend a semver bump:
      - Breaking changes → major
      - New features (`Feat:`, `feat:`) → minor
      - Fixes, refactors, docs, etc. → patch
   d. Present the detected current version, the recommended next version, and the reasoning to the user. Confirm before proceeding.

5. **Pre-release file updates (conditional):**
   a. If the project's CLAUDE.md defines a release process → follow it exactly.
   b. Otherwise, scan for files that commonly hold version or release info and propose updates for **only files that already exist in the project**. Do NOT create new files.
      - Version files (`package.json`, `pyproject.toml`, `marketplace.json`, etc.) → bump version
      - `CHANGELOG.md` or similar → add release entry based on included changes
   c. After making changes, show the user a summary of what was updated (files changed, old → new values).
   d. Commit the changes using the project's commit convention (fall back to `chore: release vX.Y.Z` if no convention is found).

6. Push the branch: `git push -u origin {branch-name}`
7. Draft the PR title and body using the Release PR Template, and create the PR: `gh pr create --base {base-branch} --assignee @me --title "..." --body "$(cat <<'EOF' ... EOF)"`. Follow the session's tool permission settings for approval.
8. Return the PR URL. Include a one-line note: **Base branch: `{base-branch}`** — {reason}. Also provide **post-merge guidance** — list the following as next steps the user should perform after merging:
   - Create and push a git tag: `git tag vX.Y.Z && git push origin vX.Y.Z`
   - Create a GitHub Release: `gh release create vX.Y.Z --generate-notes`
   - Sync the source branch back with the base branch (e.g., `git checkout {default-base} && git merge {release-base} && git push origin {default-base}`)
   - Any project-specific deployment steps described in the project's CLAUDE.md

**Important:**
- For Release PRs, automatically collect all included PRs from the merge history.
- Adapt section headers and content language to the project's CLAUDE.md language setting.
- Do NOT create files that don't already exist in the project. Only update existing files.
- Always prioritize the project's own conventions and release process over the defaults above.
