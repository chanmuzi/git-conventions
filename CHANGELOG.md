# Change Log

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
