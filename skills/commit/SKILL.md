---
name: commit
description: >-
  Create a git commit following project conventions.
  TRIGGER when: user asks to commit, create a commit, stage and commit, save changes, or any workflow that includes committing (e.g., "commit and push", "commit and create PR", "커밋해줘", "커밋하고 푸시").
  DO NOT TRIGGER when: user is only checking git status/diff without intent to commit.
version: "1.5.0"
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
| `--amend` | Amend the last commit instead of creating a new one |

Parse `$ARGUMENTS` for flags before proceeding. Natural language equivalents: "amend해줘", "직전 커밋에 합쳐줘", "마지막 커밋 수정" → `--amend`.

## Amend (opt-in only)

> Skip this section if any of the following is true:
> - `--amend` is NOT set
> - The last commit is a merge commit (`git rev-parse --verify HEAD^2 2>/dev/null` succeeds)

When `--amend` is set, amend the last commit with the current changes.

**Step 1 — Check push status:**

```bash
git log @{u}..HEAD --oneline 2>/dev/null
# - Command fails (no upstream): NOT pushed
# - Non-empty output: NOT pushed (unpushed commits exist)
# - Empty output: already pushed → force push required after amend
```

**Step 2 — Execute amend:**

- Stage the relevant files and run `git commit --amend`.
  - Use `--no-edit` if the change is a minor fix within the same intent.
  - Update the commit message if the scope or meaning has changed.

- **NOT pushed** → amend directly and display:

> **⚡ Amend** — `--amend` flag detected

- **Already pushed** → display a warning and ask for confirmation before proceeding:

> **⚠️ Amend + force push** — the last commit has already been pushed. Proceed?

  - If confirmed: amend and run `git push --force-with-lease`, then display:

> **⚡ Force pushed** (`--force-with-lease`)

  - If declined: fall through to Task (new commit).

**After amend:**
- If no unstaged or untracked changes remain → done. Do not proceed to Task.
- If any changes remain → proceed to Task for the remaining changes.

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

> If Amend already committed some or all files, only process the remaining uncommitted changes below. If nothing remains, skip this section.

1. **Intent Grouping (BEFORE staging)**:
    - List all changed files (staged + unstaged + untracked).
    - Map each file to one Intent Category (see Heuristics below).
    - Count distinct categories.
    - If 2+ distinct categories → you MUST create one commit per category. Push status (already pushed or not) MUST NOT influence this *splitting* decision. If force push becomes necessary after rewriting history, follow the same confirmation pattern as the Amend section (display the warning, ask the user, then `git push --force-with-lease`).
    - If 2+ distinct categories, display the grouping plan to the user before staging:

      ```
      **Commit Plan**
      1. {type}({scope}): {subject} — N files
         - file1, file2, ...
      2. {type}({scope}): {subject} — M files
         - file3, file4, ...
      ```

    - If only 1 category, no plan display is required — proceed directly.

2. Within each intent group, identify commit units using the Tie-breaking heuristics below. If multiple units exist within one group, plan separate commits for each.

### Commit Unit Heuristics

Apply rules in order:
1. Map each file to an Intent Category below. Files in different categories are different intents and MUST be split into separate commits.
2. Check Same-intent exceptions — keep dependent changes together even if they appear to span categories.
3. Within an intent group, use the Tie-breaking heuristics to decide whether to further split.

**Intent Categories**

Use the file's role in the project, not its filename or directory alone, to assign a category.

- `infra-deploy`   — deployment configs, environment templates, ops/setup/teardown/health/seed scripts
- `agent-meta`     — agent instructions, hooks, permission/secret guards (e.g., AGENTS.md, CLAUDE.md, .claude/, .codex/, `.gitignore` patterns guarding secrets/credentials)
- `app-runtime`    — production code implementing one feature or fix
- `build-tooling`  — build/lint/format/type-check configs, dependency manifests
- `docs`           — documentation-only changes (README, docs/, inline doc-only edits)
- `test`           — test-only additions or updates

If a file does not clearly fit, pick the category that best matches its primary effect at runtime, and note the ambiguity in the grouping plan.

**Same-intent exceptions (do NOT split)**

- schema/model change + code reading/writing the new field
- function signature change + all required call-site updates
- production code change + tests validating that exact change

**Tie-breaking heuristics (within one intent group)**

- Split work into commits that are independently revertable and cherry-pickable.
- A commit is too large if reverting it would also remove unrelated intent.
- A commit is too small if cherry-picking it would leave the codebase broken, incomplete, or missing required validation.
- Keep dependent changes in the same commit when splitting them would break a working intermediate state.
  - Example: schema/model change + code that reads or writes the new field
  - Example: function signature change + all required call-site updates
  - Example: production code change + the tests required to validate that change
- Split changes into separate commits when they have different intent or can be rolled back independently.
  - Example: refactor vs feature addition
  - Example: runtime/app behavior vs seed/tooling/docs updates
  - Example: mechanical rename/formatting vs behavioral changes
- Do not use layer boundaries alone as the rule.
  - Backend and frontend may stay together when both are required for one valid end-to-end change.
  - Backend and frontend should be split when they can be verified, reverted, or cherry-picked independently.
- When uncertain, evaluate each candidate unit with these checks:
  - If this unit alone is reverted, does it remove only one intended change?
  - If this unit alone is cherry-picked, does the result still build, run, and test coherently?
  - Are these changes driven by one reason, or are multiple motivations mixed together?
  - Would this unit make root-cause isolation easier if a regression appears later?

3. For each commit unit:
   - Stage the relevant files individually.
   - Show the proposed commit message.
   - Use a multi-line body if the commit is intentionally broad but still one cohesive unit.
   - Create the commit. Follow the session's tool permission settings for approval.
4. Repeat step 3 until all commit units are committed. Do NOT stop after the first commit — handle all units within this single skill invocation.

**Important:**
- Do NOT use `git add -A` or `git add .` — stage specific files by name.
- Do NOT include files that may contain secrets (`.env`, credentials, tokens, etc.).
