---
name: commit
description: Create a git commit following project conventions
---

## Gather Context

Run the following commands to understand the current state:
1. `git status` — check current git status
2. `git diff HEAD` — check staged/unstaged changes
3. `git branch --show-current` — check current branch
4. `git log --oneline -10` — reference recent commit style

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
- Keep it concise — aim for under 50 characters.
- Use natural, fluent phrasing. Do not force-translate well-known technical terms. For example, write `source_map` as-is rather than translating it.
- Focus on WHAT changed and WHY, not HOW.

### Examples

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

## Task

1. Analyze the diff to understand the nature and purpose of the changes.
2. Determine the appropriate type prefix (and scope, if applicable).
3. Draft a commit message following the convention above.
4. Present the proposed commit message and **wait for user approval**.
5. Once approved, stage the relevant files individually and create the commit.

**Important:**
- Do NOT commit without explicit user approval.
- Do NOT use `git add -A` or `git add .` — stage specific files by name.
- Do NOT include files that may contain secrets (`.env`, credentials, tokens, etc.).
- If there are multiple logical units of change, suggest splitting into separate commits.
