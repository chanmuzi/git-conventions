<p align="right">
  <a href="README.md">English</a> · <a href="README.ko.md">한국어</a>
</p>

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/banner-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="assets/banner-light.svg">
    <img src="assets/banner-light.svg" alt="git-claw" width="700">
  </picture>
</p>

<h1 align="center">git-claw</h1>
<p align="center">일관된 Git 워크플로우를 위한 Agent Skill</p>

<p align="center">
  <img src="https://img.shields.io/badge/skills-6-8B5CF6?logo=git&logoColor=white" alt="Skills" />
  <a href="https://agentskills.io"><img src="https://img.shields.io/badge/Agent_Skills-compatible-0EA5E9?logo=robotframework&logoColor=white" alt="Agent Skills" /></a>
  <img src="https://img.shields.io/github/license/chanmuzi/git-claw?color=blue" alt="License" />
  <img src="https://img.shields.io/github/last-commit/chanmuzi/git-claw?color=orange" alt="Last Commit" />
</p>

---

사람에게도, 에이전트에게도 읽기 좋은 Git 히스토리.

git-claw는 커밋, PR, 이슈, 코드 리뷰의 포맷을 하나로 통일하는 [Agent Skill](https://agentskills.io)입니다. 한 번 설치하면 모든 협업 산출물이 같은 컨벤션을 따릅니다 — 혼자 작업할 때도, 팀으로 협업할 때도.

- **읽기 좋은 히스토리** — 구조화된 커밋, PR, 이슈로 누구든 (어떤 에이전트든) 한눈에 맥락을 파악
- **멀티 모델 코드 리뷰** — 도메인 전문 에이전트 + Codex adversarial 분석을 교차 검증하여 severity 기반으로 정렬
- **세션 연속성** — `/handoff`로 작업 컨텍스트를 캡처해 다음 세션에서 바로 이어 작업

---

### 스킬

| | Skill | 설명 |
|:---:|---|---|
| 📝 | `/commit` | diff 기반 conventional commit 생성 |
| 🔀 | `/pr` | 구조화된 PR 생성 및 자동 labeling |
| 📋 | `/issue` | 템플릿 기반 이슈 생성 및 우선순위 labeling |
| 💬 | `/review-reply` | 리뷰 코멘트 분석 및 답글 |
| 🔍 | `/code-review` | 멀티 에이전트 severity 기반 코드 리뷰 |
| 🤝 | `/handoff` | 세션 이관 프롬프트 생성 |

## 설치

### Claude Code Plugin (권장)

```bash
# marketplace 추가
/plugin marketplace add chanmuzi/git-claw

# plugin 설치
/plugin install git-claw@git-claw
```

### Skills CLI

```bash
npx skills add chanmuzi/git-claw
```

대화형 installer에서 스킬, 대상 agent, 범위(project/global), 설치 방식을 선택할 수 있습니다.

<details>
<summary>프롬프트 없이 한번에 설치</summary>

```bash
npx skills add chanmuzi/git-claw --skill commit --skill pr --skill issue --skill review-reply --skill code-review --skill handoff -g
```

</details>

### 업데이트

**Claude Code Plugin:**

```bash
/plugin update git-claw@git-claw
```

또는 auto-update 활성화: `/plugin` → **Marketplaces** 탭 → marketplace 선택 → **Enable auto-update**

**Skills CLI:**

```bash
npx skills check    # 업데이트 확인
npx skills update   # 전체 업데이트
```

> Symlink(권장)으로 설치한 경우 한 번의 업데이트로 모든 agent에 즉시 반영됩니다. Copy로 설치한 경우 각 복사본을 개별 업데이트해야 합니다.

## 스킬

> **Agent별 호출 prefix:** Claude Code는 `/` (예: `/commit`), Codex CLI는 `$` (예: `$commit`)를 사용합니다.

### `/commit` — Git 커밋 생성

staged/unstaged 변경 사항을 분석하고, conventional commit 형식에 맞는 커밋 메시지를 제안합니다.

```
/commit              # 분석 후 커밋
/commit --amend      # 직전 커밋에 변경사항 합치기
```

타입: `feat` · `fix` · `refactor` · `style` · `docs` · `test` · `perf` · `chore` · `hotfix`

### `/pr` — Pull Request 생성

구조화된 템플릿으로 PR을 생성합니다. Individual PR과 Release PR 모드를 자동으로 구분합니다.

```
/pr              # Individual PR (Feat, Fix, Refactor 등)
/pr -g           # Mermaid 변경 흐름 그래프 포함
/pr release      # Release PR (dev → main 통합)
```

### `/issue` — GitHub 이슈 생성

구조화된 템플릿으로 이슈를 생성하고, type/priority label을 자동으로 부여합니다.

```
/issue             # 일반 이슈 (컨텍스트에서 유형 추론)
/issue bug         # Bug Report
/issue feature     # Feature Request
```

### `/review-reply` — PR 리뷰 분석 및 답글

PR의 리뷰 코멘트(CodeRabbit, Copilot, 팀원 등)를 수집하고, 실제 코드 대비 유효성을 분석한 뒤 결과를 논의합니다.

```
/review-reply          # 현재 브랜치의 PR 리뷰
/review-reply 42       # PR #42 리뷰
```

### `/code-review` — Context-Aware 코드 리뷰

PR 또는 로컬 코드 변경사항을 도메인별 전문 agent(Security, Performance, Architecture, Domain Logic)로 병렬 분석합니다. false positive를 필터링한 뒤, severity 기반 구조화된 리뷰를 생성합니다 (🔴 Critical · 🟡 Warning · 🟢 Info). 변경사항이 없으면 자동으로 현재 작업 디렉토리 리뷰로 전환하며, 대화 맥락을 반영하여 최적의 리뷰 범위를 결정합니다.

```
/code-review           # PR 자동 감지 → 변경사항 리뷰 → 현재 디렉토리 리뷰 (대화 맥락 반영)
/code-review 42        # PR #42 코드 리뷰
/code-review src/auth/ # 특정 경로 코드 리뷰
```

PR 모드에서 findings는 GitHub Review API를 통해 diff의 특정 라인에 inline comment로 게시됩니다.

<details>
<summary>플래그</summary>

- `--wd` — PR 자동 감지를 건너뛰고 working directory 모드 강제
- `--domain security,perf` — 자동 감지 대신 도메인 수동 지정
- `-y` / `-f` — 승인 없이 즉시 게시
- `-g` — Mermaid 변경 흐름 그래프 생성 (PR 모드 전용)
- `-q` / `--quick` — Quick 모드: 단일 패스, 도메인 최대 2개, Critical/Warning 우선
- `--no-codex` — Codex 통합 비활성화
- `--codex-both` — Codex 일반 리뷰 + adversarial 동시 실행

</details>

<details>
<summary>Codex 통합 (선택 사항)</summary>

[Codex 플러그인](https://github.com/openai/codex) 설치 및 인증(`claude plugin add codex` + `!codex setup`) 환경에서 `/code-review` 실행 시 Codex adversarial review가 자동 병렬 실행됩니다. findings는 교차 검증 후 출처 태그와 함께 통합됩니다. `--no-codex`로 비활성화 가능합니다.

</details>

### `/handoff` — 세션 Handoff 프롬프트

현재 세션의 작업 컨텍스트를 다음 세션으로 이관하기 위한 copy-ready handoff 프롬프트를 생성합니다. 아티팩트, git 상태, 대화 컨텍스트를 자동 감지하고 **참조 우선(reference-not-repeat)** 원칙에 따라 구성합니다.

```
/handoff                   # 자동 감지 후 handoff 프롬프트 생성
/handoff -y                # 확인 없이 즉시 출력
/handoff auth 리팩토링      # 특정 주제에 집중한 handoff 생성
```

<details>
<summary>상세</summary>

**감지 cascade:** 아티팩트 (`.omc/specs/`, `.omc/plans/`) → Git 상태 → 대화 컨텍스트

**스킬 추천:** OMC, Codex 등의 플러그인이 감지되면 다음 세션에 가장 적합한 스킬을 추천합니다 (예: `/autopilot`, `/ralph`, `/commit`).

**출력:** `/copy`에 최적화된 터미널 텍스트. 아티팩트 내용을 반복하지 않고 파일 경로만 참조합니다.

</details>

## 언어 동작

모든 커맨드의 출력(커밋 메시지, PR 제목/본문)은 프로젝트의 `CLAUDE.md`에 설정된 언어로 작성됩니다. 설정이 없으면 사용자의 대화 언어를 따릅니다. 기술 용어는 원어 그대로 유지합니다.

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

## Label 시스템

`/pr`과 `/issue`는 통일된 label 체계를 공유합니다. Label은 최초 사용 시 자동 생성되며, 이미 존재하면 그대로 유지됩니다.

<details>
<summary>Label 목록</summary>

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

</details>

## 라이선스

MIT
