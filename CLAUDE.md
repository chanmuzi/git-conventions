# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

Git commit, PR, review 관련 convention을 강제하는 Agent Skill.
`/commit`, `/pr`, `/pr release`, `/issue`, `/review-reply`, `/code-review` 여섯 가지 slash command를 제공한다.
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
│   ├── issue/
│   │   ├── SKILL.md                  # /issue [bug|feature|...]
│   │   └── LICENSE.txt
│   ├── review-reply/
│   │   ├── SKILL.md                  # /review-reply
│   │   └── LICENSE.txt
│   └── code-review/
│       ├── SKILL.md                  # /code-review
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
/issue         → 이슈 템플릿 생성 및 label 자동 부여 확인
/issue bug     → Bug Report 템플릿 확인
/review-reply        → AI 리뷰 코멘트 수집/분석 확인
/review-reply 42     → 특정 PR 번호로 리뷰 확인
/code-review         → PR 자동 감지 또는 working directory 리뷰
/code-review 42      → 특정 PR 코드 리뷰
/code-review src/    → 특정 경로 코드 리뷰
/code-review --wd    → PR 브랜치에서도 working dir 리뷰 강제
```

## Git Convention (이 프로젝트 자체에 적용)

- Commit: `{type}: {한글 설명}` (소문자 prefix)
- Type: feat, fix, refactor, style, docs, test, perf, chore, hotfix
- 파일 staging 시 `git add -A` 금지, 개별 파일 지정
- 커밋 승인은 세션의 tool permission 설정에 따름
- PR merge 시 squash 금지 — 커밋 히스토리를 보존하여 agent/reviewer 추적성 유지

## 스킬 공통 규칙

모든 스킬(pr, issue, review-reply, code-review)에 적용되는 공통 원칙:

- **Assignee**: `gh pr create`, `gh issue create` 시 반드시 `--assignee @me` 포함
- **참조 라벨 구분**: "관련" 표기 시 유형을 명시 — `관련 커밋:` (SHA), `관련 PR:` (#번호). 단독 `관련:`은 사용 금지
- **Bullet 관리**: 카테고리당 bullet 5개 초과 시 관련 항목을 통합하여 가독성 유지
- **Commit SHA 표기**: backtick 금지 (GitHub 링크화 방지됨). plain text 또는 markdown link 사용

## 새 스킬 추가 체크리스트

새 스킬 디렉토리(`skills/{name}/`) 생성 시 아래 파일을 반드시 함께 업데이트한다:

1. `skills/{name}/SKILL.md` — 스킬 본문 (frontmatter 포함)
2. `skills/{name}/LICENSE.txt` — 라이선스 파일
3. `.claude-plugin/marketplace.json` — `skills` 배열에 경로 추가, `description` 반영
4. `CLAUDE.md` — 프로젝트 개요(slash command 목록), 아키텍처 트리, 로컬 테스트, 스킬 공통 규칙
5. `README.md` — 스킬 소개 섹션, 설치 명령어(`--skill`), CLAUDE.md 연동 예시
6. `README.ko.md` — README.md와 동일 항목 한국어 반영
7. `CHANGELOG.md` — 변경 항목 추가

위 목록 반영 후, 기존 스킬 이름(예: `review-reply`)으로 Grep하여 참조가 있는 파일을 추가 확인한다. 누락된 파일이 있으면 함께 업데이트한다.

## 릴리스 프로세스

새 버전 배포 시 다음 항목을 업데이트한다:

1. `.claude-plugin/marketplace.json` — `metadata.version` bump
2. `skills/*/SKILL.md` — frontmatter `version` 동기화
3. `CHANGELOG.md` — Change Log 항목 추가
4. `README.md`, `README.ko.md` — 필요 시 업데이트 (새 기능/변경된 동작)

## 언어

문서, 주석, commit 메시지 등은 한글 중심으로 작성. 고유 기술 용어(plugin, frontmatter, slash command 등)는 영어 유지.
SKILL.md 내용은 Claude가 읽는 지시문이므로 영어로 작성.

## Change Log

[CHANGELOG.md](./CHANGELOG.md) 참조.
