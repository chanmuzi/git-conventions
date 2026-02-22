# git-conventions

A Claude Code skill that enforces consistent Git commit messages, PR templates, and AI review analysis across projects.

<details>
<summary>🇺🇸 English Version</summary>

## Installation

```bash
# Add marketplace
/plugin marketplace add chanmuzi/git-conventions

# Install plugin
/plugin install git-conventions@git-conventions
```

## Skills

### `/commit` — Create a Git Commit

Analyzes your staged/unstaged changes and proposes a commit message following the conventional commit format.

```
/commit
```

**Commit message format:**
```
{type}: {description}
{type}({scope}): {description}
```

Types: `feat`, `fix`, `refactor`, `style`, `docs`, `test`, `perf`, `chore`, `hotfix`

### `/pr` — Create a Pull Request

Creates a PR with a structured template. Automatically detects whether to use the Individual PR template or the Release PR template.

```
/pr              # Individual PR (Feat, Fix, Refactor, etc.)
/pr release      # Release PR (dev → main integration)
```

**PR title format:**
```
{Type}: {description}               # Individual (e.g., Feat: add new feature)
Release: dev → main 통합 (vX.Y.Z)  # Release
```

### `/review` — Review AI PR Comments

Collects AI-generated review comments (CodeRabbit, Copilot, etc.) from a PR, analyzes their validity against the actual code, and discusses findings with you.

```
/review          # Review current branch's PR
/review 42       # Review PR #42
```

## Language Behavior

All commands write output (commit messages, PR titles/body) in the language configured in your project's `CLAUDE.md`. If no language is set, the user's conversational language is used.

Technical terms are kept in their original form to preserve clarity — no forced translations.

## Conventions at a Glance

| Item | Format | Example |
|------|--------|---------|
| Commit | `{type}: {desc}` (lowercase) | `feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Branch | `{type}/{kebab-case}` (English) | `feat/multiturn-context-persistence` |
| PR Title | `{Type}: {desc}` (capitalized) | `Feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Release PR | `Release: dev → main 통합 (vX.Y.Z)` | `Release: dev → main 통합 (v0.4.1)` |

## Integration with CLAUDE.md

Add the following to your global `~/.claude/CLAUDE.md` to reference these conventions:

```markdown
## Git Conventions
- Commit: `{type}: {description}` (lowercase prefix: feat, fix, refactor, style, docs, test, perf, chore, hotfix)
- Branch: `{type}/{english-kebab-case}` (feat/, fix/, refactor/, docs/, hotfix/)
- PR title: `{Type}: {description}` (capitalized prefix: Feat, Fix, Refactor, Perf, etc.)
- Release PR: `Release: dev → main 통합 (vX.Y.Z)`
- Use `/commit`, `/pr`, `/pr release`, `/review` commands for full workflows
```

## License

MIT

</details>

<details>
<summary>🇰🇷 한국어 버전</summary>

## 설치

```bash
# marketplace 추가
/plugin marketplace add chanmuzi/git-conventions

# plugin 설치
/plugin install git-conventions@git-conventions
```

## 스킬

### `/commit` — Git 커밋 생성

staged/unstaged 변경 사항을 분석하고, conventional commit 형식에 맞는 커밋 메시지를 제안합니다.

```
/commit
```

**커밋 메시지 형식:**
```
{type}: {설명}
{type}({scope}): {설명}
```

타입: `feat`, `fix`, `refactor`, `style`, `docs`, `test`, `perf`, `chore`, `hotfix`

### `/pr` — Pull Request 생성

구조화된 템플릿으로 PR을 생성합니다. Individual PR 템플릿과 Release PR 템플릿을 자동으로 구분합니다.

```
/pr              # Individual PR (Feat, Fix, Refactor 등)
/pr release      # Release PR (dev → main 통합)
```

**PR 제목 형식:**
```
{Type}: {설명}                      # Individual (예: Feat: 새 기능 추가)
Release: dev → main 통합 (vX.Y.Z)  # Release
```

### `/review` — AI PR 리뷰 분석

PR의 AI 리뷰 코멘트(CodeRabbit, Copilot 등)를 수집하고, 실제 코드 대비 유효성을 분석한 뒤 결과를 논의합니다.

```
/review          # 현재 브랜치의 PR 리뷰
/review 42       # PR #42 리뷰
```

## 언어 동작

모든 커맨드의 출력(커밋 메시지, PR 제목/본문)은 프로젝트의 `CLAUDE.md`에 설정된 언어로 작성됩니다. 설정이 없으면 사용자의 대화 언어를 따릅니다.

기술 용어는 명확성을 위해 원어 그대로 유지합니다.

## Convention 요약

| 항목 | 형식 | 예시 |
|------|------|------|
| Commit | `{type}: {설명}` (소문자) | `feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Branch | `{type}/{kebab-case}` (영어) | `feat/multiturn-context-persistence` |
| PR 제목 | `{Type}: {설명}` (대문자) | `Feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Release PR | `Release: dev → main 통합 (vX.Y.Z)` | `Release: dev → main 통합 (v0.4.1)` |

## CLAUDE.md 연동

글로벌 `~/.claude/CLAUDE.md`에 아래 내용을 추가하면 convention을 참조할 수 있습니다:

```markdown
## Git Conventions
- Commit: `{type}: {description}` (소문자 prefix: feat, fix, refactor, style, docs, test, perf, chore, hotfix)
- Branch: `{type}/{english-kebab-case}` (feat/, fix/, refactor/, docs/, hotfix/)
- PR title: `{Type}: {description}` (대문자 prefix: Feat, Fix, Refactor, Perf 등)
- Release PR: `Release: dev → main 통합 (vX.Y.Z)`
- `/commit`, `/pr`, `/pr release`, `/review` 커맨드로 전체 워크플로우 실행
```

## 라이선스

MIT

</details>
