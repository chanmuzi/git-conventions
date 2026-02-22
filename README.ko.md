# git-conventions

[English](README.md) | [한국어](README.ko.md)

Git commit, PR, review convention을 강제하는 [Agent Skill](https://agentskills.io)입니다.

Claude Code, Codex CLI, Gemini CLI, Cursor, GitHub Copilot, Amp 등 [40개 이상의 agent](https://skills.sh)에서 사용할 수 있습니다.

## 설치

### 방법 1: Skills CLI (모든 Agent 공용)

```bash
npx skills add chanmuzi/git-conventions
```

대화형 설치가 진행됩니다:

1. **Select skills to install** — `space`로 설치할 skill 선택
   - `commit` — 프로젝트 convention에 맞는 git commit 생성
   - `pr` — 프로젝트 convention에 맞는 pull request 생성
   - `review` — AI PR 리뷰 코멘트 분석
2. **Which agents do you want to install to?** — 사용할 agent 선택 (예: Codex, Cursor, Gemini CLI, GitHub Copilot, …)
3. **Installation scope** — `Project` (현재 repo만) 또는 `Global` (모든 프로젝트에서 사용)
4. **Installation method** — `Symlink` (권장) 또는 `Copy`
5. **확인** 후 설치 완료

> 프롬프트 없이 한번에 설치하려면:
> ```bash
> npx skills add chanmuzi/git-conventions --skill commit --skill pr --skill review -g
> ```

### 방법 2: Claude Code Plugin

```bash
# marketplace 추가
/plugin marketplace add chanmuzi/git-conventions

# plugin 설치
/plugin install git-conventions@git-conventions
```

## 업데이트

### Skills CLI

```bash
# 업데이트 가능 여부 확인
npx skills check

# 설치된 모든 skill 업데이트
npx skills update
```

> Symlink(권장)으로 설치한 경우 한 번의 업데이트로 모든 agent에 즉시 반영됩니다. Copy로 설치한 경우 각 복사본을 개별 업데이트해야 합니다.

### Claude Code Plugin

```bash
# plugin 업데이트
/plugin update git-conventions@git-conventions
```

또는 auto-update를 활성화하면 자동으로 최신 버전이 적용됩니다:

`/plugin` → **Marketplaces** 탭 → marketplace 선택 → **Enable auto-update**

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
