<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:6366F1,50:8B5CF6,100:A855F7&height=220&section=header&text=git-claw&fontSize=52&fontColor=ffffff&fontAlignY=38&desc=Agent%20Skills%20for%20consistent%20Git%20workflows&descSize=18&descColor=E2E8F0&descAlignY=58&animation=fadeIn" alt="git-claw" width="100%" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/skills-6-8B5CF6?logo=git&logoColor=white" alt="Skills" />
  <a href="https://agentskills.io"><img src="https://img.shields.io/badge/Agent_Skills-compatible-0EA5E9?logo=robotframework&logoColor=white" alt="Agent Skills" /></a>
  <img src="https://img.shields.io/github/license/chanmuzi/git-claw?color=blue" alt="License" />
  <img src="https://img.shields.io/github/last-commit/chanmuzi/git-claw?color=orange" alt="Last Commit" />
</p>

<p align="center">
  <a href="README.md">English</a> · <a href="README.ko.md">한국어</a>
</p>

---

Git commit, PR, review convention을 강제하는 [Agent Skill](https://agentskills.io)입니다.

Claude Code, Codex CLI, Gemini CLI, Cursor, GitHub Copilot, Amp 등 [40개 이상의 agent](https://skills.sh)에서 사용할 수 있습니다.

## 설치

### 방법 1: Claude Code Plugin (권장)

```bash
# marketplace 추가
/plugin marketplace add chanmuzi/git-claw

# plugin 설치
/plugin install git-claw@git-claw
```

### 방법 2: Skills CLI (모든 Agent 공용)

```bash
npx skills add chanmuzi/git-claw
```

**포함된 스킬:**

| Skill | 설명 |
|-------|------|
| `commit` | 프로젝트 convention에 맞는 git commit 생성 |
| `pr` | 프로젝트 convention에 맞는 pull request 생성 |
| `issue` | 템플릿 기반 GitHub 이슈 생성 및 자동 labeling |
| `review-reply` | PR 리뷰 코멘트 분석 및 답글 |
| `code-review` | context-aware multi-agent 코드 리뷰 (severity 기반) |
| `handoff` | 세션 이관을 위한 copy-ready handoff 프롬프트 생성 |

대화형 installer에서 스킬, 대상 agent, 범위(project/global), 설치 방식을 선택할 수 있습니다.

> 프롬프트 없이 한번에 설치하려면:
> ```bash
> npx skills add chanmuzi/git-claw --skill commit --skill pr --skill issue --skill review-reply --skill code-review --skill handoff -g
> ```

## 업데이트

> **권장:** auto-update를 활성화하면 새로운 기능과 수정 사항을 자동으로 받을 수 있습니다.
> 아래 플랫폼별 안내를 참고하세요.

### Claude Code Plugin

```bash
# plugin 업데이트
/plugin update git-claw@git-claw
```

또는 auto-update를 활성화하면 자동으로 최신 버전이 적용됩니다 (권장):

`/plugin` → **Marketplaces** 탭 → marketplace 선택 → **Enable auto-update**

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

## 스킬

> **Agent별 호출 prefix:** Claude Code는 `/` (예: `/commit`), Codex CLI는 `$` (예: `$commit`)를 사용합니다.

### `/commit` — Git 커밋 생성

staged/unstaged 변경 사항을 분석하고, conventional commit 형식에 맞는 커밋 메시지를 제안합니다.

```
/commit              # 분석 후 커밋
/commit --amend      # 직전 커밋에 현재 변경사항 합치기 (amend)
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
/pr -g           # Mermaid 변경 흐름 그래프 포함
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
**자동 labeling:** `bug`, `feature`, `enhancement`, `critical`/`high`/`medium`/`low` 등

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
- `-y` / `-f` — 승인 없이 즉시 게시
- `-g` — Mermaid 변경 흐름 그래프 생성 (PR 모드 전용)
- `-q` / `--quick` — Quick 모드: 단일 패스 분석 (에이전트 미사용), 도메인 최대 2개, Critical/Warning 우선 (없으면 Info fallback)
- `--no-codex` — Codex 통합 비활성화
- `--codex` — Codex 강제 활성화 (adversarial only, default와 동일)
- `--codex-general` — Codex 일반 리뷰만 사용 (adversarial 없이)
- `--codex-both` — Codex 일반 리뷰 + adversarial 동시 실행

**자연어 지원:** 플래그 또는 자연어 모두 사용 가능합니다 — Claude가 의도를 해석하여 적절한 플래그로 변환합니다.

```
/code-review 42 바로 올려줘             # → /code-review 42 -y
/code-review security만 봐줘            # → /code-review --domain security
/code-review codex 둘 다 돌려줘         # → /code-review --codex-both
/code-review codex 없이 리뷰해줘        # → /code-review --no-codex
/code-review working dir로 봐줘         # → /code-review --wd
/code-review 간단하게 봐줘              # → /code-review --quick
```

**Codex 통합 (선택 사항, 권장):** [Codex 플러그인](https://github.com/openai/codex) 설치 및 인증 환경에서 `/code-review` 실행 시 Codex adversarial review가 도메인 에이전트와 자동 병렬 실행됩니다. findings는 교차검증 후 출처 태그와 함께 통합 정렬됩니다. `--no-codex`로 비활성화 가능합니다.

Codex 통합 설정:
1. Codex 플러그인 설치: `claude plugin add codex`
2. Claude Code에서 `!codex setup` 실행하여 OpenAI API key 인증
3. `/code-review` 실행 시 Codex 자동 탐지 및 사용 — 추가 플래그 불필요

**Inline review comments:** PR 모드에서 findings는 GitHub Review API를 통해 diff의 특정 라인에 inline review comment로 게시됩니다. 각 finding이 독립 스레드로 생성되어 개별 답변, resolve, suggestion 적용이 가능합니다. review body에는 severity 요약 테이블이 포함되며, diff 밖의 findings는 "General Findings"로 요약에 포함됩니다.

**Severity:** 🔴 Critical, 🟡 Warning, 🟢 Info

### `/handoff` — 세션 Handoff 프롬프트

현재 세션의 작업 컨텍스트를 다음 세션으로 이관하기 위한 copy-ready handoff 프롬프트를 생성합니다. 아티팩트, git 상태, 대화 컨텍스트를 자동 감지하고 **참조 우선(reference-not-repeat)** 원칙에 따라 간결한 프롬프트를 구성합니다.

```
/handoff                   # 자동 감지 후 handoff 프롬프트 생성
/handoff -y                # 확인 없이 즉시 출력
/handoff auth 리팩토링      # 특정 주제에 집중한 handoff 생성
```

**감지 cascade:** 아티팩트 (`.omc/specs/`, `.omc/plans/`) → Git 상태 → 대화 컨텍스트

**스킬 추천:** OMC, Codex 등의 플러그인이 감지되면 다음 세션에 가장 적합한 스킬을 추천합니다 (예: `/autopilot`, `/ralph`, `/commit`).

**출력:** `/copy`에 최적화된 터미널 텍스트. 아티팩트 내용을 반복하지 않고 파일 경로만 참조합니다.

## 언어 동작

모든 커맨드의 출력(커밋 메시지, PR 제목/본문)은 프로젝트의 `CLAUDE.md`에 설정된 언어로 작성됩니다. 설정이 없으면 사용자의 대화 언어를 따릅니다.

기술 용어는 명확성을 위해 원어 그대로 유지합니다.

## Label 시스템

`/pr`과 `/issue`는 통일된 label 체계를 공유합니다. Label은 최초 사용 시 자동 생성되며, 이미 존재하면 그대로 유지됩니다.

| Label | Color | 출처 |
|-------|-------|------|
| `bug` | 🔴 `d73a4a` | GitHub 기본 |
| `feature` | 🔵 `0075ca` | GitHub 기본 |
| `enhancement` | 🩵 `a2eeef` | GitHub 기본 |
| `docs` | 🟣 `5319e7` | GitHub 기본 |
| `chore` | 🟡 `e4e669` | 표준 |
| `refactor` | 🟪 `d4c5f9` | 표준 |
| `test` | 🟢 `bfd4f2` | 표준 |
| `perf` | 🟠 `f9d0c4` | 표준 |
| `hotfix` | 🔴 `b60205` | 표준 |
| `release` | 🔵 `1d76db` | 표준 |
| `critical` | 🔴 `b60205` | 우선순위 |
| `high` | 🟠 `d93f0b` | 우선순위 |
| `medium` | 🟡 `fbca04` | 우선순위 |
| `low` | 🟢 `0e8a16` | 우선순위 |

- **Namespace 접두사 없음** — 깔끔하고 간결한 이름 (`type: bug` 대신 `bug`)
- **GitHub 표준 색상** — GitHub 기본 palette과 동일한 색상 사용
- **충돌 안전** — `gh label create ... 2>/dev/null || true` (없으면 생성, 있으면 스킵)

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
- `/commit`, `/pr`, `/pr release`, `/issue`, `/review-reply`, `/code-review`, `/handoff` 커맨드로 전체 워크플로우 실행
```

## 라이선스

MIT
