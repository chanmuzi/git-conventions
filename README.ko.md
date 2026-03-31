<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:6366F1,50:8B5CF6,100:A855F7&height=220&section=header&text=git-conventions&fontSize=52&fontColor=ffffff&fontAlignY=38&desc=Agent%20Skills%20for%20consistent%20Git%20workflows&descSize=18&descColor=E2E8F0&descAlignY=58&animation=fadeIn" alt="git-conventions" width="100%" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/skills-5-8B5CF6?logo=git&logoColor=white" alt="Skills" />
  <a href="https://agentskills.io"><img src="https://img.shields.io/badge/Agent_Skills-compatible-0EA5E9?logo=robotframework&logoColor=white" alt="Agent Skills" /></a>
  <img src="https://img.shields.io/github/license/chanmuzi/git-conventions?color=blue" alt="License" />
  <img src="https://img.shields.io/github/last-commit/chanmuzi/git-conventions?color=orange" alt="Last Commit" />
</p>

<p align="center">
  <a href="README.md">English</a> · <a href="README.ko.md">한국어</a>
</p>

---

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
   - `issue` — 템플릿 기반 GitHub 이슈 생성 및 자동 labeling
   - `review-reply` — PR 리뷰 코멘트 분석 및 답글
   - `code-review` — context-aware multi-agent 코드 리뷰 (severity 기반)
2. **Which agents do you want to install to?** — 사용할 agent 선택 (예: Codex, Cursor, Gemini CLI, GitHub Copilot, …)
3. **Installation scope** — `Project` (현재 repo만) 또는 `Global` (모든 프로젝트에서 사용)
4. **Installation method** — `Symlink` (권장) 또는 `Copy`
5. **확인** 후 설치 완료

> 프롬프트 없이 한번에 설치하려면:
> ```bash
> npx skills add chanmuzi/git-conventions --skill commit --skill pr --skill issue --skill review-reply --skill code-review -g
> ```

### 방법 2: Claude Code Plugin

```bash
# marketplace 추가
/plugin marketplace add chanmuzi/git-conventions

# plugin 설치
/plugin install git-conventions@git-conventions
```

## 업데이트

> **권장:** auto-update를 활성화하면 새로운 기능과 수정 사항을 자동으로 받을 수 있습니다.
> 아래 플랫폼별 안내를 참고하세요.

### Skills CLI

```bash
# 업데이트 가능 여부 확인
npx skills check

# 설치된 모든 skill 업데이트
npx skills update
```

> Symlink(권장)으로 설치한 경우 한 번의 업데이트로 모든 agent에 즉시 반영됩니다. Copy로 설치한 경우 각 복사본을 개별 업데이트해야 합니다.
>
> Skills CLI는 auto-update를 지원하지 않습니다. `npx skills check`를 주기적으로 실행하거나, [changelog](CHANGELOG.md)에 공지된 주요 릴리스 후 업데이트하세요.

### Claude Code Plugin

```bash
# plugin 업데이트
/plugin update git-conventions@git-conventions
```

또는 auto-update를 활성화하면 자동으로 최신 버전이 적용됩니다 (권장):

`/plugin` → **Marketplaces** 탭 → marketplace 선택 → **Enable auto-update**

## 스킬

> **Agent별 호출 prefix:** Claude Code는 `/` (예: `/commit`), Codex CLI는 `$` (예: `$commit`)를 사용합니다.

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

### `/issue` — GitHub 이슈 생성

구조화된 템플릿으로 이슈를 생성하고, type/priority label을 자동으로 부여합니다.

```
/issue             # 일반 이슈 (컨텍스트에서 유형 추론)
/issue bug         # Bug Report
/issue feature     # Feature Request
```

**템플릿:** Bug Report, Feature Request, General
**자동 labeling:** `type: bug`, `type: feature`, `type: enhancement`, `priority: critical/high/medium/low` 등

### `/review-reply` — PR 리뷰 분석 및 답글

PR의 리뷰 코멘트(CodeRabbit, Copilot, 팀원 등)를 수집하고, 실제 코드 대비 유효성을 분석한 뒤 결과를 논의합니다.

```
/review-reply          # 현재 브랜치의 PR 리뷰
/review-reply 42       # PR #42 리뷰
```

### `/code-review` — Context-Aware 코드 리뷰

PR 또는 로컬 코드 변경사항을 도메인별 전문 agent(Security, Performance, Architecture, Domain Logic)로 병렬 분석합니다. 코드 context와 대조하여 false positive를 필터링한 뒤, severity 기반 구조화된 리뷰를 생성합니다.

```
/code-review           # 현재 브랜치의 PR 자동 감지, 없으면 working dir 리뷰
/code-review 42        # PR #42 코드 리뷰
/code-review src/auth/ # 특정 경로 코드 리뷰
/code-review --wd      # PR 브랜치에서도 working dir 리뷰 강제
```

**플래그:**
- `--wd` — PR 자동 감지를 건너뛰고 working directory 모드 강제
- `--domain security,perf` — 자동 감지 대신 도메인 수동 지정
- `--inline` — PR에 inline comment 추가 (PR 모드 전용)
- `-y` / `-f` — 승인 없이 즉시 게시
- `-g` — 코드 그래프 분석 활성화
- `--no-codex` — Codex 통합 비활성화
- `--codex-review` — adversarial 대신 Codex 일반 리뷰 사용
- `--codex-both` — Codex 일반 리뷰 + adversarial 동시 실행

**자연어 지원:** 플래그 또는 자연어 모두 사용 가능합니다 — Claude가 의도를 해석하여 적절한 플래그로 변환합니다.

```
/code-review 42 바로 올려줘             # → /code-review 42 -y
/code-review security만 봐줘            # → /code-review --domain security
/code-review codex 둘 다 돌려줘         # → /code-review --codex-both
/code-review codex 없이 리뷰해줘        # → /code-review --no-codex
/code-review working dir로 봐줘         # → /code-review --wd
```

**Codex 통합:** [Codex 플러그인](https://github.com/anthropics/codex) 설치 환경에서 `/code-review` 실행 시 Codex adversarial review가 도메인 에이전트와 자동 병렬 실행됩니다. findings는 교차검증 후 출처 태그와 함께 통합 정렬됩니다. `--no-codex`로 비활성화 가능합니다.

**Severity:** 🔴 Critical, 🟡 Warning, 🟢 Info

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
- `/commit`, `/pr`, `/pr release`, `/issue`, `/review-reply`, `/code-review` 커맨드로 전체 워크플로우 실행
```

## 라이선스

MIT
