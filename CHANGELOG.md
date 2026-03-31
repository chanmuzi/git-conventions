# Change Log

### 2026-04-01: 터미널 렌더링 가이드라인 추가

- review-reply 템플릿 항목 간 빈 줄 → `---` 변경 (터미널에서 numbered list 내 빈 줄이 무시되는 문제 수정)
- CLAUDE.md에 "터미널 렌더링 가이드라인" 섹션 신설 — Markdown 요소별 터미널 렌더링 차이 테이블, 항목 분리 원칙

### 2026-03-31: label 간소화, README 개선 (v1.7.1)

- Label에서 `type:`, `priority:` namespace 접두사 제거 — `bug`, `feature`, `critical` 등 간결한 이름으로 변경
- README에 Label System 섹션 추가 (GitHub 표준 색상, 충돌 안전 자동 생성)
- 설치/업데이트 섹션 순서 변경 — Claude Code Plugin을 Recommended로 격상
- Skills CLI 설치 섹션 가독성 개선 (스킬 목록 테이블 분리)
- 새 스킬 추가 체크리스트에 `skills-N` 배지 count 동기화 항목 추가

### 2026-03-31: /code-review inline review comments 기본 전환 (v1.7.0)

**Inline Review Comments (code-review v1.7.0)**
- PR 모드에서 findings를 GitHub Review API를 통해 inline review comment로 게시 (기본 동작)
- 각 finding이 diff의 특정 라인에 개별 스레드로 생성 — reply, resolve, suggestion 적용 가능
- Review body에 severity 요약 테이블 포함, diff 밖 findings는 "General Findings"로 요약에 포함
- GitHub `suggestion` 블록 지원: 구체적 코드 수정은 one-click apply 가능
- `--inline` 플래그 제거 — inline이 기본 동작으로 전환됨
- domain agent findings에 primary line number 필수 포함 (inline comment 타겟팅용)
- Fallback: Review API 실패 시 기존 summary comment 방식으로 자동 전환
- review-reply 스킬과 완전 호환 — inline comment가 `pulls/comments` API로 조회됨

### 2026-03-31: /code-review Codex adversarial-review 통합 (v1.6.0)

**Codex 멀티모델 리뷰 통합 (code-review v1.6.0)**
- Codex 플러그인 설치 환경에서 adversarial-review 자동 병렬 실행 (opt-out 방식)
- zero-cost 감지: system prompt의 available skills 목록으로 Codex 유무 판별 (hooks/환경변수 불필요)
- Codex 플래그 추가: `--no-codex` (비활성화), `--codex-review` (일반 리뷰), `--codex-both` (리뷰 + adversarial 동시)
- Step 2.5 Codex Detection 단계 신규 추가
- Step 3에서 Codex agent를 도메인 에이전트와 동일 레벨로 병렬 실행 (Team/Sub-agent 모드 지원)
- Step 4 Cross-Validation에 Codex findings 합류 — 동일 품질 검증 적용
- Step 5 출처 태그: default → `— Codex`, `--codex-both` → `— Codex` + `— Codex Adv`
- Codex 없는 환경에서 기존 동작 100% 동일 (하위호환)

### 2026-03-31: /code-review PR 자동 탐지 및 --wd 플래그 추가

**PR Auto-Detection (code-review v1.5.0)**
- 인자 없이 `/code-review` 실행 시 현재 브랜치의 열린 PR 자동 감지
- `--author @me` 필터로 봇 PR (dependabot, renovate 등) 자동 스킵
- `--wd` 플래그 추가: PR 브랜치에서도 Working Dir 모드 강제 가능
- Mode Detection을 우선순위 기반 5단계로 확장

### 2026-03-31: /code-review 출력 포맷 이중 분리 (Terminal / GitHub)

**Output Template 이중 포맷 (code-review v1.4.0)**
- 터미널 포맷: CLI 가독성 최적화 — 요약 테이블 제거, 플랫 구조, blockquote Fix
- GitHub 포맷: PR 코멘트 최적화 — summary 테이블, `<details>` 접기, diff 블록 (구체적 수정 시)
- PR 모드 2-phase 플로우: 터미널 미리보기 → 승인 → GitHub 포맷으로 게시
- Finding 구조 개선: 파일 위치를 첫 줄로 분리, 제목에서 파일명 중복 제거, 라인 번호 `L42` 형식 통일

### 2026-03-31: /code-review 스킬 추가 — context-aware multi-agent 코드 리뷰

**Code Review 스킬 신규 추가 (code-review)**
- `/code-review` slash command 신규 생성
- 3가지 모드 지원: PR Review (`/code-review 42`), Working Dir (`/code-review`), Path Review (`/code-review src/`)
- 4개 도메인 agent 병렬 실행: Security, Performance, Architecture, Domain Logic
- 파일 타입 기반 도메인 자동 활성화 (override: `--domain`)
- Cross-validation 단계로 false positive 필터링 (git history, 주석, PR description 대조)
- Severity-first 구조화 출력 포맷 (🔴 Critical / 🟡 Warning / 🟢 Info)
- 플래그: `--inline` (PR inline comment), `-y`/`-f` (즉시 게시), `-g` (코드 그래프)
- Agent tool 미지원 runner에서 sequential fallback 제공

**문서 업데이트**
- CLAUDE.md: 아키텍처 트리, 테스트 예시, 스킬 공통 규칙에 code-review 반영
- marketplace.json: skills 배열 및 description 업데이트
- README.md, README.ko.md: 스킬 소개, 설치 명령어, CLAUDE.md 연동 섹션 업데이트

### 2026-03-30: argument-hint 제거, review → review-reply 리네임, GraphQL shell-safe 패턴 적용

**전체 스킬 공통**
- `argument-hint` frontmatter 제거 (pr, issue, review-reply): 불필요한 입력 안내가 오히려 혼란을 주어 삭제

**Review 스킬 리네임 (review → review-reply)**
- skill name `review` → `review-reply`로 변경: "코드 리뷰 수행"이 아닌 "리뷰 코멘트에 답글" 역할을 명확히 반영
- 디렉토리 `skills/review/` → `skills/review-reply/`로 이동
- marketplace.json, CLAUDE.md, README.md, README.ko.md 참조 일괄 업데이트

**GraphQL shell-safe 패턴 적용 (review-reply)**
- `gh api graphql -f query='...'` → heredoc + `--input -` 패턴으로 변경
- GraphQL 변수의 `$`가 shell에서 치환되어 빈 문자열이 되는 문제 해결
- review thread ID 수집(query)과 thread resolve(mutation) 두 곳 모두 적용

### 2026-03-30: Skills 2.0 frontmatter 확장 (v1.3.0)

**전체 스킬 공통**
- `allowed-tools` frontmatter 추가: 스킬별 사용 가능 도구를 명시적으로 제한
  - commit: `Bash(git *)`, Read, Grep, Glob
  - pr: `Bash(git *)`, `Bash(gh pr *)`, `Bash(gh label *)`, Read, Grep, Glob
  - issue: `Bash(gh issue *)`, `Bash(gh label *)`, Read, Grep, Glob
  - review: `Bash(gh *)`, `Bash(git *)`, Read, Grep, Glob
- `argument-hint` frontmatter 추가 (pr, issue, review): 슬래시 커맨드 자동완성 힌트 제공
- version 1.2.0 → 1.3.0

### 2026-03-30: PR 참조 라벨 명확화 및 bullet 관리 가이드라인 추가

**PR 템플릿 개선 (pr)**
- "관련:" → "관련 커밋:" / "관련 PR:"로 유형 명시 강제
- 카테고리당 bullet 5개 초과 시 통합 가이드라인 추가
- Individual PR 템플릿에 per-category 참조 패턴 추가 (커밋/PR 예시)
- Assignee 누락 방지를 위한 Important 섹션 강조 추가

**Issue 스킬 개선 (issue)**
- Assignee 누락 방지를 위한 Important 섹션 강조 추가

**CLAUDE.md**
- "스킬 공통 규칙" 섹션 신규 추가: assignee, 참조 라벨, bullet 관리, SHA 표기 규칙 문서화

### 2026-03-29: issue skill 추가, label 체계 도입, review reply 개선 (v1.2.0)

**Issue 스킬 신규 추가 (issue)**
- `/issue` slash command 신규 생성
- Bug Report / Feature Request / General 3종 템플릿 제공
- 이슈 유형에 따른 type label 자동 부여
- priority label 지원 (사용자 지정 시)

**Label 체계 도입 (pr, issue 공통)**
- type label 9종 + priority label 4종, 색상 코드 통일
- PR은 title prefix에서, issue는 내용/인자에서 label 자동 매핑
- label 미존재 시 자동 생성 (`gh label create`)

**PR 스킬 개선 (pr)**
- Commit history 보존 원칙 추가: 사용자 요청 없이 squash/rebase/amend 금지
- `gh pr create`에 `--label` 플래그 추가

**Review 스킬 개선 (review)**
- Reply 포맷 구조화: bold status headline + bullet points 강제
- "Resolve applied threads" → "Resolve concluded threads"로 명확화
- Resolve rules 테이블 추가 (Applied/Won't Fix → resolve, Follow-up → 유지)
- resolve 즉시 실행 강제 지시 추가

### 2026-03-29: review 미반영 항목 Won't Fix / Follow-up 분기 (v1.1.6)

**Review 스킬 개선 (review)**
- 미반영 항목을 Won't Fix / Follow-up 두 유형으로 분기
  - Won't Fix: reply에 사유 기록 + thread resolve (issue 생성 안 함)
  - Follow-up: reply에 사유 + 이슈 번호 기록 + issue 생성 (thread는 unresolved 유지)
- Reply 포맷 3종 분리: ✅ 반영 / ⏭️ Won't Fix / 🔜 Follow-up
- 후속 이슈 제목 변경: "미반영 피드백 정리" → "후속 작업"

### 2026-03-29: review thread resolve 및 후속 이슈 생성 지원 (v1.1.5)

**Review 스킬 개선 (review)**
- 반영 완료 thread 자동 resolve: GraphQL API로 conversation 닫기
- 미반영 항목 후속 이슈 생성: 관련 항목끼리 묶어서 consolidated issue 생성 (1건당 1이슈 금지)
- Step 4에 deferred 마킹 단계 추가
- Step 1에 review thread ID 수집 (GraphQL) 추가

### 2026-03-29: issue comment 수집 및 답글 지원 (v1.1.4)

**Review 스킬 개선 (review)**
- issue comment 수집 추가: CodeRabbit 등 AI 리뷰어가 issue comment로 포스팅하는 경우 대응
- 답글 API 분기: review comment(reply API) vs. issue comment(새 comment + 원본 인용) 구분 처리
- comment source type 추적 지시 추가

### 2026-03-29: PR 템플릿 경량화 및 의도 중심 구조 개편 (v1.1.3)

**PR 템플릿 개선 (pr)**
- 개요 섹션 강화: 1-3문장 → 2-4문장, 배경→동기→접근 방식 구조로 변경
- 파일 변경 요약 테이블 제거: GitHub diff와 중복되는 정보 삭제
- 테스트 섹션 제거: 실사용에서 활용도 낮은 섹션 삭제
- 참고 사항 섹션 신규 추가: 리뷰 포인트, 의도적 미작업, 후속 작업 등 결정 맥락 기록
- Release PR: 배포 정보 테이블 제거, 참고 사항으로 대체
- 이모지 헤더 제거: 가독성 및 에이전트 파싱 개선

### 2026-03-03: 스킬 자동 트리거 및 복수 커밋 지원 (v1.1.2)

**TRIGGER 조건 추가 (commit, pr, review)**
- description에 TRIGGER/DO NOT TRIGGER 조건 명시
- 자연어 요청("커밋해줘", "PR 올려줘" 등)에서 스킬 자동 트리거 개선

**복수 커밋 처리 (commit)**
- 여러 논리 단위가 있을 때 한 번의 스킬 호출 안에서 전부 커밋하도록 Task 변경
- 기존: "분리를 제안하라" → 변경: "각 단위별로 stage → commit을 반복하라"

**PR assignee 자동 지정 (pr)**
- `gh pr create`에 `--assignee @me` 추가

### 2026-03-03: PR 스킬 이중 승인 제거 (v1.1.1)

**PR Task 단계 통합 (pr)**
- "초안 보여주기"(step 2)와 "PR 생성"(step 5) 분리 → 한 단계로 통합
- 세션의 tool permission 설정에 위임 (commit 스킬과 동일 패턴)
- 기존 flow에서 발생하던 이중 승인(수동 확인 + Bash 권한) 문제 해결

**Change Log 분리**
- CLAUDE.md에서 CHANGELOG.md로 이관
- 릴리스 프로세스에서 참조 경로 변경

### 2026-02-26: 승인 flow 세션 권한 위임 및 Gather Context 조건부 실행

**승인 flow 세션 권한 위임 (commit, pr)**
- Task 단계에서 "present → 승인 대기" 분리 제거, 한 단계로 통합
- 세션의 tool permission 설정에 자연스럽게 위임

**Gather Context 조건부 실행 (commit)**
- 대화 컨텍스트에서 변경 사항을 이미 알고 있으면 `git status`/`git diff HEAD` 생략

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
