---
name: pr
description: >-
  Create a pull request following project conventions.
  TRIGGER when: user asks to create/open a PR, push and create PR, or any workflow that includes PR creation (e.g., "PR 올려줘", "푸시하고 PR 만들어줘", "commit, push, and create PR").
  DO NOT TRIGGER when: user is viewing or listing existing PRs, or performing git operations without PR intent.
argument-hint: "[release] [--base <branch>] [-g|--graph]"
version: "1.3.1"
allowed-tools: Bash(git *), Bash(gh pr *), Bash(gh label *), Read, Grep, Glob
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

## Flag Detection

| Flag | Default | Description |
|------|---------|-------------|
| `-g\|--graph` | off | Include a Mermaid change-flow graph in the PR description |

## Gather Context

Run the following commands using the resolved `{base-branch}`:
1. `git log --oneline {base-branch}..HEAD 2>/dev/null || git log --oneline -10` — list commits on this branch
2. `git diff {base-branch}...HEAD --stat 2>/dev/null || echo "Could not determine diff"` — list changed files

### Graph Analysis (when `-g` flag is set)

Additionally build a **change-flow graph** for the PR description:

1. **Trace relationships**: For each changed file, identify imports, exports, function calls, and data flow to/from other changed files using Grep.
2. **Map edges**: Record directional relationships — code-level (`imports`, `calls`, `extends`, `emits/consumes`) and conceptual (`references`, `shared logic`, `configures`).
3. **Group by module**: If changed files exceed 7, group by parent directory into logical modules (subgraph nodes). Individual files become child nodes.
4. **Construct Mermaid source**: Build a `flowchart LR` diagram. Nodes = changed files (or module groups). Edges = relationships found in step 2, labeled with relationship type.

**Skip condition**: If no edges (relationships) are found between changed files after step 2, skip graph generation entirely. Omit the graph section from the PR description silently.

Graph rules:
- Mermaid direction: `flowchart LR` (left-to-right).
- Node labels: filename only (no full path). Use `[filename.ext]` format (basename + extension).
- Edge labels: relationship type — code-level (`imports`, `calls`, `extends`, `emits/consumes`, `reads/writes`) or conceptual (`references`, `shared logic`, `configures`).
- Module grouping: When changed files > 7, group by parent directory using `subgraph`. When ≤ 7, show individual file nodes without subgraph.

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

## Label System

Assign a type label to the PR based on its title prefix. This label scheme is shared with the `/issue` skill for project-wide consistency. The core labels are identical; `/issue` additionally defines `enhancement` for improvement requests.

### Type Labels

| Label | Color | PR type prefix |
|-------|-------|----------------|
| `feature` | `0075ca` | `Feat:` |
| `bug` | `d73a4a` | `Fix:` |
| `refactor` | `d4c5f9` | `Refactor:` |
| `perf` | `f9d0c4` | `Perf:` |
| `docs` | `5319e7` | `Docs:` |
| `test` | `bfd4f2` | `Test:` |
| `chore` | `e4e669` | `Chore:` |
| `hotfix` | `b60205` | `Hotfix:` |
| `release` | `1d76db` | `Release:` |

### Priority Labels (optional — assign only if the user specifies)

| Label | Color |
|-------|-------|
| `critical` | `b60205` |
| `high` | `d93f0b` |
| `medium` | `fbca04` |
| `low` | `0e8a16` |

Before assigning a label, ensure it exists in the repository. Color values are 6-character hex without `#`:
```
gh label create "{label}" --color "{hex}" 2>/dev/null || true
```

---

## Individual PR Template

Apply this template for feature, fix, refactor, and other non-release PRs.

### Formatting guidance

- Prefer bullet points over prose paragraphs. Each bullet should be one clear, concise statement.
- Keep "개요" to 1-2 sentences.
- For single-concern PRs, use a flat bullet list under "변경 사항" without sub-headings.
- Do NOT put commit SHAs, PR numbers, or other references in heading lines (`###`). Place them as body text below the heading — headings should contain only descriptive titles.
- **Bullet management**: When a category exceeds 5 bullets, consolidate related items into fewer, broader bullets. Combine closely related changes with commas (e.g., "해커톤 필터 개선, 검색 위치 조정, 캘린더 네비게이션 수정"). Aim for ≤5 bullets per category.
- **Reference labeling**: Always qualify per-category references with the type — use `관련 커밋:` for commit SHAs, `관련 PR:` for PR numbers. Never use bare `관련:`.

```markdown
## 개요

{1-2 sentence summary: what this PR does and why}

> ⚠️ **Breaking Change**: {only if applicable — describe migration needed}

## 변경 사항

### 1. {Change category title}

- {Specific change and its purpose}
- {Specific change and its purpose}
- 관련 커밋: {sha1}, {sha2}

### 2. {Change category title}

- {Specific change and its purpose}
- 관련 PR: #{number}


{if -g flag set AND relationships found:}
## 변경 흐름

```mermaid
flowchart LR
  subgraph {module_name}[{Module Display Name}]
    {file_node_id}[{filename}]
  end
  {source_node} -->|{relationship_label}| {target_node}
```

{end if}

## 참고 사항

- {Points reviewers should focus on}
- {Intentional omissions and reasons}
- {Follow-up work if any}

(Omit this section entirely if nothing noteworthy.)

## 관련 이슈/PR

- #{number}
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

### 1. {Feature/Fix name}

- {Key change}
- {Key change}
- 관련 PR: #{PR number}

### 2. {Feature/Fix name}

- {Key change}
- 관련 PR: #{PR number}


{if -g flag set AND relationships found:}
## 변경 흐름

```mermaid
flowchart LR
  subgraph {module_name}[{Module Display Name}]
    {file_node_id}[{filename}]
  end
  {source_node} -->|{relationship_label}| {target_node}
```

{end if}

## 참고 사항

- {Deployment considerations}
- {Follow-up work if any}

(Omit this section entirely if nothing noteworthy.)

## 관련 PR

- #{number1}
- #{number2}
```

---

## Task

1. **Resolve base branch** following the "Determine Base Branch" rules above.
2. Determine PR type: Release (if `$ARGUMENTS` contains "release") or Individual.
3. Sync check before pushing:
   - `git fetch origin {base-branch}`
   - If the local branch is behind, inform the user and suggest an appropriate action (rebase, merge, or proceed as-is).

### If Individual PR:

4. Push the branch if not already pushed: `git push -u origin {branch-name}`. If push fails due to permission denied, run `gh repo fork --remote` and retry the push. If forking fails, inform the user with the specific error and stop. If push fails for other reasons, inform the user with the specific error and stop.
5. Ensure the type label exists: `gh label create "{type_label}" --color "{hex}" 2>/dev/null || true`
6. Draft the PR title and body using the Individual PR Template, and create the PR: `gh pr create --base {base-branch} --assignee @me --label "{type_label}" --title "..." --body "$(cat <<'EOF' ... EOF)"`. If the command fails due to `--assignee` or `--label` permissions, retry without those flags. Follow the session's tool permission settings for approval.
7. Return the PR URL. Include a one-line note: **Base branch: `{base-branch}`** — {reason} (e.g., "CLAUDE.md Branch Strategy 설정에 따라 결정" or "origin/dev에서 분기한 브랜치로 감지").

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

6. Push the branch: `git push -u origin {branch-name}`. If push fails due to permission denied, run `gh repo fork --remote` and retry. If forking fails, inform the user with the specific error and stop. If push fails for other reasons, inform the user with the specific error and stop.
7. Ensure the type label exists: `gh label create "release" --color "1d76db" 2>/dev/null || true`
8. Draft the PR title and body using the Release PR Template, and create the PR: `gh pr create --base {base-branch} --assignee @me --label "release" --title "..." --body "$(cat <<'EOF' ... EOF)"`. If the command fails due to `--assignee` or `--label` permissions, retry without those flags. Follow the session's tool permission settings for approval.
9. Return the PR URL. Include a one-line note: **Base branch: `{base-branch}`** — {reason}. Also provide **post-merge guidance** — list the following as next steps the user should perform after merging:
   - Create and push a git tag: `git tag vX.Y.Z && git push origin vX.Y.Z`
   - Create a GitHub Release: `gh release create vX.Y.Z --generate-notes`
   - Sync the source branch back with the base branch (e.g., `git checkout {default-base} && git merge {release-base} && git push origin {default-base}`)
   - Any project-specific deployment steps described in the project's CLAUDE.md

**Important:**
- Do NOT modify commit history (squash, rebase, amend, reorder) before pushing unless the user explicitly requests it. Preserve all commits to maintain traceable context for agents and reviewers.
- For Release PRs, automatically collect all included PRs from the merge history.
- Adapt section headers and content language to the project's CLAUDE.md language setting.
- Do NOT create files that don't already exist in the project. Only update existing files.
- Always prioritize the project's own conventions and release process over the defaults above.
- **Assignee**: Always include `--assignee @me` in `gh pr create`. If it fails due to insufficient permissions, retry without it.
- **Commit references**: Never wrap commit SHAs in backticks (e.g., `` `abc1234` ``). Backtick-wrapped SHAs render as inline code and are not clickable on GitHub. Use plain text (GitHub auto-links SHAs) or explicit markdown links: `[{short_sha}](https://github.com/{owner}/{repo}/commit/{sha})`.
