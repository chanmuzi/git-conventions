---
name: commit
description: >-
  Create a git commit following project conventions.
  TRIGGER when: user asks to commit, create a commit, stage and commit, save changes, or any workflow that includes committing (e.g., "commit and push", "commit and create PR", "커밋해줘", "커밋하고 푸시").
  DO NOT TRIGGER when: user is only checking git status/diff without intent to commit.
version: "1.3.0"
allowed-tools: Bash(git *), Read, Grep, Glob
---

## Gather Context

If the current changes are already known from the conversation context (e.g., you just edited the files), skip these commands.
Otherwise, run them to understand the current state:
1. `git status` — check current status and untracked files
2. `git diff HEAD` — review all changes

## Flag Detection

| Flag | Effect |
|------|--------|
| `--no-amend` | Skip amend detection; always create a new commit |

Parse `$ARGUMENTS` for flags before proceeding. Natural language equivalents: "새 커밋으로", "amend 하지 말고" → `--no-amend`.

## Amend Detection

> Skip this section if any of the following is true:
> - `--no-amend` is set
> - This is the very first commit (`HEAD~1` does not exist)
> - The last commit is a merge commit (`git rev-parse --verify HEAD^2 2>/dev/null` succeeds)

Before creating a new commit, check whether the current changes should amend the last commit.

**Step 1 — Check file overlap:**

Run both commands and compare the file lists:
```bash
git diff HEAD~1 HEAD --name-only      # files changed in the last commit
git diff HEAD --name-only             # files currently modified (vs HEAD)
```

If **no file appears in both lists** (intersection is empty) → proceed to Task (new commit).
If **any file** appears in both lists → proceed to Step 2 (amend).

**Step 2 — Amend with push-status awareness:**

If files overlap, amend the last commit:

```bash
# Check whether the last commit has been pushed
git log @{u}..HEAD --oneline 2>/dev/null
# - Command fails (no upstream): NOT pushed
# - Non-empty output: NOT pushed (unpushed commits exist)
# - Empty output: already pushed → force push required after amend
```

- Stage the relevant files and run `git commit --amend`.
  - Use `--no-edit` if the change is a minor fix within the same intent.
  - Update the commit message if the scope or meaning has changed.
- Display a hint:

> **⚡ Amend** — overlapping files with the last commit detected

- If the last commit was already pushed, also run `git push --force-with-lease` and display:

> **⚡ Force pushed** (`--force-with-lease`)

**After amend:**
- If all changed files were covered by the amend → done. Do not proceed to Task.
- If some files were NOT in the last commit → proceed to Task for remaining files only.

## Commit Message Convention

Format: `{type}: {description}` or `{type}({scope}): {description}`

### Type Prefixes (always lowercase)

| Type | Purpose |
|------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code restructuring without behavior change |
| `style` | Formatting, linting (no logic change) |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `perf` | Performance improvement |
| `chore` | Build, CI, dependencies, config |
| `hotfix` | Urgent production fix |

### Scope (optional)

Add scope in parentheses when the change targets a specific module or component:
- `feat(source-map): ...`
- `fix(full-text): ...`
- `test(query-rewriting): ...`

### Description Rules

- Write the description in the language configured in the project's CLAUDE.md. If no language is configured, follow the user's conversational language.
- Keep the subject line concise — aim for under 50 characters.
- Use natural, fluent phrasing. Do not force-translate well-known technical terms. For example, write `source_map` as-is rather than translating it.
- Focus on WHAT changed and WHY, not HOW.

### Multi-line Body (for complex changes)

When the diff involves multiple files or logical units, add a body after a blank line to explain details:

```
{type}: {subject line}

- {detail 1}
- {detail 2}
- {detail 3}
```

**When to use multi-line:**
- 3+ files changed across different concerns
- Non-obvious reasoning behind the change
- Breaking changes or migration notes

**When single-line is enough:**
- Simple, self-explanatory changes (typo fix, single-file edit, etc.)

### Examples

**Single-line:**
```
feat: 멀티턴 컨텍스트 유지 기능 추가
fix: 요청 DTO 모델 기본값을 settings 기반으로 통일
refactor: 미사용 openai_api_key/openai_base_url 프로퍼티 제거
style: black 포맷팅 적용
perf: Classification max_tokens 30 → 15로 조정
chore: 보안 취약점 패키지 업데이트 (urllib3, langchain-core)
docs: README 환경변수 가이드 업데이트
test: 멀티턴 컨텍스트 유지 기능 테스트 추가
feat(source-map): 검색하지 않은 턴에서도 source_map 지속 반환
fix(full-text): 전문 조회 실패 시 state 누출 방지
hotfix: uv lock synced
```

**Multi-line:**
```
refactor: Gather Context 불필요한 명령 및 중복 step 제거

- commit: git branch, git log 명령 삭제 (gitStatus/컨벤션 정의로 대체)
- pr: git remote show origin, gh pr list 네트워크 호출 제거
- review: Gather Context 섹션 삭제, 카테고리 불일치 수정
```

## Task

> If Amend Detection already committed some or all files, only process the remaining uncommitted changes below. If nothing remains, skip this section.

1. Analyze the diff and identify logical units of change. If there are multiple units, plan separate commits for each.
2. For each commit unit:
   - Stage the relevant files individually.
   - Show the proposed commit message.
   - Create the commit. Follow the session's tool permission settings for approval.
3. Repeat step 2 until all logical units are committed. Do NOT stop after the first commit — handle all units within this single skill invocation.

**Important:**
- Do NOT use `git add -A` or `git add .` — stage specific files by name.
- Do NOT include files that may contain secrets (`.env`, credentials, tokens, etc.).
