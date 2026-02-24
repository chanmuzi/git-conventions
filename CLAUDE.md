# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

Git commit, PR, review 관련 convention을 강제하는 Agent Skill.
`/commit`, `/pr`, `/pr release`, `/review` 세 가지 slash command를 제공한다.
Agent Skills 오픈 표준(agentskills.io)을 따르며, Claude Code 외에도 Codex CLI, Gemini CLI, Cursor 등 40+ agent에서 사용 가능.

## 아키텍처

Claude Code skill 표준 구조(`anthropics/skills` 기준)를 따른다.

```
git-conventions/
├── .claude-plugin/marketplace.json   # marketplace 등록 메타데이터
├── skills/
│   ├── commit/
│   │   ├── SKILL.md                  # /commit
│   │   └── LICENSE.txt
│   ├── pr/
│   │   ├── SKILL.md                  # /pr, /pr release
│   │   └── LICENSE.txt
│   └── review/
│       ├── SKILL.md                  # /review [PR번호]
│       └── LICENSE.txt
├── .gitignore
├── README.md
└── LICENSE
```

- 각 SKILL.md는 YAML frontmatter(`name`, `description`)로 시작
- SKILL.md body에서 Claude가 직접 git/gh 명령을 실행하도록 지시
- `$ARGUMENTS`로 사용자 인자를 참조

## 배포

두 가지 설치 방식을 지원한다:

- **Skills CLI (크로스 플랫폼)**: `npx skills add chanmuzi/git-conventions`
- **Claude Code Plugin**: `/plugin marketplace add chanmuzi/git-conventions`

## 로컬 테스트

```bash
# 방법 1: Skills CLI
npx skills add ./  # 로컬 경로에서 설치

# 방법 2: Claude Code Plugin
/plugin marketplace add /Users/chanmuzi/coding/workspace/git-conventions
/plugin install git-conventions@git-conventions

# 각 command 테스트
/commit        → diff 분석 후 conventional commit 메시지 제안 확인
/pr            → Individual PR 템플릿 생성 확인
/pr release    → Release PR 템플릿 생성 확인
/review        → AI 리뷰 코멘트 수집/분석 확인
/review 42     → 특정 PR 번호로 리뷰 확인
```

## Git Convention (이 프로젝트 자체에 적용)

- Commit: `{type}: {한글 설명}` (소문자 prefix)
- Type: feat, fix, refactor, style, docs, test, perf, chore, hotfix
- 파일 staging 시 `git add -A` 금지, 개별 파일 지정
- 커밋 전 반드시 사용자 승인 필요

## 릴리스 프로세스

새 버전 배포 시 다음 항목을 업데이트한다:

1. `.claude-plugin/marketplace.json` — `metadata.version` bump
2. `skills/*/SKILL.md` — frontmatter `version` 동기화
3. `CLAUDE.md` — Change Log 항목 추가
4. `README.md`, `README.ko.md` — 필요 시 업데이트 (새 기능/변경된 동작)

## 언어

문서, 주석, commit 메시지 등은 한글 중심으로 작성. 고유 기술 용어(plugin, frontmatter, slash command 등)는 영어 유지.
SKILL.md 내용은 Claude가 읽는 지시문이므로 영어로 작성.

## Change Log

### 2026-02-25: 승인 flow 개선 및 version 메타데이터 도입 (v1.1.0)

**승인 flow 개선 (commit, pr)**
- SKILL.md 내 명시적 "wait for user approval" 제거
- Claude의 tool permission 시스템에 위임 (auto-edits: 자동 진행, 수동: bash 실행 시 1회 확인)
- review는 interactive flow 유지 (변경 없음)

**Version 메타데이터**
- 3개 SKILL.md frontmatter에 `version` 필드 추가 (agentskills.io 표준)
- marketplace.json version 1.0.0 → 1.1.0

**릴리스 프로세스 문서화**
- CLAUDE.md에 릴리스 시 업데이트 항목 가이드 추가

**PR 동기화 개선**
- `/pr` push 전 `git fetch` + base branch 동기화 확인 단계 추가

**README 업데이트 섹션 보강**
- auto-update 설정 권장, Skills CLI 주기적 체크 안내

### 2026-02-23: Gather Context 최적화 및 README 업데이트 섹션 추가

**스킬 개선 (3개 스킬 공통)**
- 불필요한 shell 명령 제거: 전체 12개 → 6개로 축소
  - `git branch --show-current` 제거 (gitStatus에서 이미 제공)
  - `git log --oneline -10` 제거 (commit 컨벤션이 SKILL.md에 이미 정의됨)
  - `git remote show origin`, `gh pr list --state merged` 제거 (네트워크 호출 불필요)
- Task step 간소화: 중복 분석 단계 합치기
- `/review`: Gather Context 섹션 삭제, 카테고리 불일치 수정 (4개→3개), reply 언어 유연화

**README 업데이트**
- README.md, README.ko.md에 업데이트 가이드(Skills CLI / Claude Code Plugin) 추가
