# Change Log

## Unreleased

**Fixed**
- Codex UI에서 `/review-reply` 스킬이 `review`로 노출되던 메타데이터 불일치 수정 — `agents/openai.yaml`의 `display_name`을 실제 스킬명과 동기화
- `/code-review` 도메인 에이전트 spawn 시 외부 agent type 사용 방지 — prompt-only 방식 명시적 강제
- `/pr` `--assignee`/`--label` 권한 fallback 시 silent drop 방지 — 누락된 플래그를 사용자에게 명시적으로 고지하여 수동 보완 가능하도록 변경

**New**
- `/pr`, `/issue` `Style:` type / `style` label 추가 — 포맷팅·공백·코드 스타일 변경을 위한 type prefix 및 label (`c5def5`)
- `/review-reply` 카테고리별 고유 prefix (V/D/X) — 교차 참조 시 모호성 제거
- `/code-review` PR Purpose Injection — 도메인 에이전트에 PR 목적을 전달하여 목적 기반 누락/불완전 감지
- `/code-review` Out-of-Diff Causality Filter — diff 외부 finding에 인과관계 판정 적용, pre-existing issue 자동 제외
- `/code-review` `--full-scan` 플래그 — PR과 무관한 기존 이슈도 General Findings에 포함
- `/code-review` Context-Aware Scope Adjustment — 대화 맥락 기반 리뷰 범위 자동 결정 및 변경사항 없을 때 코드베이스 리뷰 fallback

**Changed**
- `/handoff` Next-action relevance 원칙 추가 — 이번 세션에서 완료된 작업이 다음 작업의 entry precondition이 아니면 handoff에서 제외하도록 강제. Layer 2(Conversation Context Analysis)를 "이번 세션 회고"가 아닌 "다음 세션이 행동하기 위해 필요한 정보 추출" 관점으로 재정의하고, exclusion criteria(무관한 머지된 PR/리네임/브랜딩 변경, NOT_PLANNED 사이드 이슈, 회고형 enumeration)를 명시. `상황` 섹션 정의를 "current state, background"에서 "entry preconditions"로 정밀화. Anti-patterns 5종 도입 및 출력 직전 self-check 단계("이 bullet 빼면 다음 세션이 어디서 막히는가") 추가. README.md/README.ko.md handoff 섹션에 두 원칙(next-action relevance + reference-not-repeat) 명시
- `/commit` 커밋 분리 기준 구체화 — `logical unit` 추상 표현을 `revert`/`cherry-pick` 기준과 의존성 휴리스틱으로 보강
- `/commit` README 설명 보강 — 필요 시 독립적인 commit unit으로 분리한다는 동작을 명시
- `/commit` Intent Grouping 단계 추가 (commit skill v1.5.0) — staging 전 변경 파일을 6종 카테고리(`infra-deploy`/`agent-meta`/`app-runtime`/`build-tooling`/`docs`/`test`)로 그룹핑하고, 2+ 카테고리 시 카테고리당 별도 commit 강제. Push 상태(이미 push 여부)가 분할 결정에 영향 주지 않도록 명시. Force push가 필요한 경우 Amend 섹션과 동일한 confirmation 패턴 적용
- `/commit` Same-intent exception 명시 — schema+code, function signature+call-sites, production code+validating tests는 같은 commit으로 유지
- `/commit` `.gitignore` 카테고리 명시 — secret/credential 가드 패턴이 포함된 `.gitignore`는 `agent-meta` 카테고리로 분류 (단순 build ignore는 `build-tooling`)
- `/commit` Intent Classification 강제력 강화 — 분류 단계를 메시지 작성 전(Amend 다음, Convention 앞)으로 이동, 파일→카테고리 매핑 enumeration을 카테고리 수와 무관하게 항상 출력하도록 무조건 강제, Same-intent exceptions를 closed whitelist로 고정, 자의적 통합을 유발하는 anti-patterns 목록 신설(주제 일관성/feature session/PR 설명에 의존 등). Step 3가 `Commit Plan`(파일→카테고리 → exceptions 적용 → tie-breaking 적용 후 commit unit 목록)을 산출하도록 정리하고, pre-commit invariant 및 post-commit self-check 모두 raw category count가 아닌 `Commit Plan`의 unit 수를 기준으로 검증하도록 통일. Worked examples 4종(straightforward / exception-merge / tie-break-split / combined) 수록하여 동일 클래스의 내부 모순 재발 방지

**Docs**
- README.md / README.ko.md / CLAUDE.md에 Codex(Skills CLI) 동기화 절차 및 stale skill cleanup 안내 추가 — `npx skills`가 자동 갱신·rename 추적을 하지 않아 옛 디렉터리(`~/.codex/skills/{old-name}/`)가 잔존하는 문제(예: `review` → `review-reply` rename 후 옛 `review` skill이 동시 노출되는 현상) 대응
- CLAUDE.md Git Convention에 main push 후 Codex 동기화 명령(`npx skills update -g chanmuzi/git-claw`) 및 rename PR의 CHANGELOG cleanup 안내 의무화 명시

---

## v1.8.0 (2026-04-05)

새 스킬 `/handoff` 도입, `/code-review` 아키텍처 전환(OMC → prompt-only), `/commit` amend opt-in 전환, Claw'd 브랜드 배너 도입, 터미널 출력 포맷 전면 개선.

**New**
- `/handoff` 스킬 신규 추가 — 세션 이관 프롬프트 생성 (PR #10, #11, #15)
- `-g`/`--graph` Mermaid 변경 흐름 그래프 — code-review, pr (PR #9)
- `/code-review` Quick 모드 `--quick` / `-q` (PR #19)
- `/code-review` Info finding에 Recommendation 행동 지침 (PR #18)
- `/commit` `--amend` 플래그 (PR #14)
- `--codex-both` 플래그 신규 (PR #23)
- 외부 레포 권한 부족 시 graceful fallback (PR #5)
- Claw'd 브랜드 배너 SVG — dark/light mode 자동 전환 (PR #25)

**Changed**
- `/code-review` OMC 의존성 제거, prompt-only 도메인 에이전트 전환 (PR #21)
- `/commit` amend 기본값 반전: 자동 감지(opt-out) → 명시적 요청(opt-in) (PR #24)
- `--codex` 플래그 의미 변경: both → 강제 활성화 (PR #23)
- 터미널 출력 Unicode 위계 체계 도입 (PR #6, #8)
- review-reply 출력 포맷 가독성 개선 (PR #8)
- README Hero Zone 도입 및 구조 개편 (PR #26)

**Fixed**
- Codex Detection을 find 기반 결정적 탐지로 전환 (PR #17, #22)
- `/handoff` Detection 순서 수정 및 `/copy` 호환성 (PR #11)
- `/commit` initial commit에서 `--amend` 무시되는 문제 (PR #24)

---

### 2026-04-05: README 내용 구조 개편

**docs**
- Hero Zone 도입: Value Proposition 문구 + 차별화 포인트 3-bullet + 스킬 테이블 요약
- 언어 전환 링크를 상단 우측(`align="right"`)으로 이동하여 Hero 흐름 유지
- Update 섹션을 Installation 하위(`###`)로 통합
- Label System을 하단(License 위)으로 이동
- `/code-review` 상세 축소: 자연어 예시 블록 제거, Codex 설정·플래그를 `<details>`로 접힘 처리
- `/handoff` 상세(감지 cascade, 스킬 추천)를 `<details>`로 접힘 처리
- Label 테이블을 `<details>`로 접힘 처리
- `/commit` 커밋 메시지 형식 블록 제거, 타입 한 줄 나열로 간소화
- README.ko.md 동기화

---

### 2026-04-05: Claw'd 브랜드 배너 도입

**assets**
- Claw'd 캐릭터 + Git 브랜치 시각화 배너 SVG 추가 (dark/light)
- 독립 Claw'd 캐릭터 SVG 추가 (점프, 눈깜빡, 팔흔들기 애니메이션)

**docs**
- README.md, README.ko.md 헤더 배너를 capsule-render → 로컬 SVG로 교체
- `<picture>` 태그로 GitHub dark/light mode 자동 전환

---

### 2026-04-04: /commit amend 기본값 반전 — opt-out → opt-in

**commit**
- `--no-amend` 플래그 제거, `--amend` 플래그로 대체 — 기본 동작이 항상 새 커밋 생성으로 변경
- 파일 겹침 기반 자동 amend 감지 제거 — 예측 불가능한 히스토리 변경 방지
- `--amend` 사용 시 push 상태 확인: 미push → 즉시 amend, push됨 → 경고 후 사용자 확인
- 자연어 매핑 업데이트: "amend해줘", "직전 커밋에 합쳐줘", "마지막 커밋 수정" → `--amend`

**docs**
- CLAUDE.md 테스트 케이스 업데이트
- README.md, README.ko.md `/commit` 섹션 동기화

---

### 2026-04-04: /code-review Codex 플래그 의미 정정

**code-review**
- `--codex` 플래그 의미 변경: "both (review + adversarial)" → "강제 활성화 (adversarial only, default와 동일)"
- `--codex-both` 플래그 신규 추가: review + adversarial 병렬 실행 (기존 `--codex` 동작)
- `--codex` + `--codex-general` 조합 시 **both** 모드 (= `--codex-both`)로 동작
- Mode Resolution 테이블 7단계로 확장 (기존 6단계)
- Source Tags 테이블 플래그 표기 동기화

**docs**
- CLAUDE.md, README.md, README.ko.md 플래그 설명 및 자연어 예시 동기화

---

### 2026-04-04: /code-review Codex companion 감지 안정성 강화

**code-review**
- `find | sort -V | tail -1` → `find -print | sort | tail -1`로 변경: `sort -V` (version sort)가 macOS 기본 `sort`(BSD sort)에서 미지원되어 불안정했던 문제 해결
- Step 2.5, Step 3 두 곳 모두 동일하게 수정
- Companion Detection을 명령형 gate 패턴으로 재작성: 실행 → 평가 → 진행 순서를 강제하여 Claude가 감지를 건너뛰는 문제 방지

---

### 2026-04-04: /code-review OMC 의존성 제거, prompt-only 도메인 에이전트 전환

**code-review**
- OMC(`oh-my-claudecode:*`) subagent_type 의존성 완전 제거 — OMC 미설치 환경에서 Error throw로 리뷰 실패하던 문제 해결
- 4개 도메인 에이전트(Security, Performance, Architecture, Domain Logic)를 prompt-only 방식으로 전환
- 각 도메인에 구조화된 Investigation Protocol 도입: 단계별 분석 절차 + Evidence Gate (근거 없는 finding 차단)
- Security: OWASP 기반 8단계 프로토콜 (secrets, injection, auth/authz, crypto, dependency CVE, CORS/CSP, logging)
- Performance: 6단계 프로토콜 (algorithmic complexity, N+1, async blocking, memory, caching, resource management)
- Architecture: 6단계 프로토콜 (pattern consistency, SOLID, coupling, API contracts, module boundaries, tech debt)
- Domain Logic: 7단계 프로토콜 (intent verification, business rules, error handling, edge cases, race conditions, type safety, state management)
- Common Prompt Suffix: Constraints (read-only, 언어 매칭, no manufactured findings), Severity Criteria 3단계, Output Format 필드 명세
- 에이전트 spawn 시 `model: "opus"` 명시
- Fallback 지시 단순화: "subagent types are not registered" 조건 제거

---

### 2026-04-02: /code-review Quick 모드 추가

**code-review**
- `--quick` / `-q` 플래그 추가: 단일 패스 분석 모드 (에이전트 spawn 없음)
- Quick 모드 시 auto-detect 도메인을 우선순위(Security > Domain Logic > Architecture > Performance) 기반 최대 2개로 cap
- Codex, Graph(`-g`) 자동 비활성화
- Critical/Warning 존재 시 Info 생략, 없으면 Info fallback 표시
- Cross-Validation 경량화: context 15줄 + sanity check만 수행 (git history, PR description 교차 생략)
- 개발/테스트 시 빠른 반복을 위한 용도
- Terminal format zero-findings 메시지 추가 (Quick/Normal 공통: `✅ No issues found.`)

---

### 2026-04-02: /code-review Info finding에 Recommendation 추가

**code-review**
- Info severity finding에 `> **Recommendation**: {Accept | Monitor | Won't Fix} — {reason}` 행동 지침 추가
- Terminal, GitHub Inline Comment, GitHub General Findings 전 포맷에 severity별 분기 적용
- Critical/Warning은 기존 `> **Fix**:` 유지, Info는 `> **Recommendation**:` 사용
- Formatting Rules 업데이트: terminal-specific, GitHub-specific 규칙에 severity별 action line 명시
- Recommendation Labels 정의 테이블 추가: Accept (조치 불필요), Monitor (향후 주시), Won't Fix (수정 비용 > 이점)
- Domain Agent 지시문의 "suggested fix"를 severity-aware "action line per severity"로 수정
- 브랜치 테스트 가이드 개선: `ls -td` 최신 cache 경로 탐색 방식으로 전환 (installed_plugins.json ≠ 실제 로드 경로 문제 해결)
- cache 누적 정리 가이드 추가 (reload마다 새 hash 생성되어 누적됨)
- `/reload-plugins`는 이 세션에서 실행해야 한다는 안내 추가
- 테스트 안내 흐름: Claude가 복사·실행·리포트·복원 전담, 사용자는 reload만

---

### 2026-04-02: /code-review Codex Detection 개선

**code-review**
- Codex 탐지 방식 전환: LLM introspection → `find` 기반 companion path 단일 게이트 (결정적 탐지)
- 플래그 이름 변경: `--codex-both` → `--codex` (full), `--codex-review` → `--codex-general` (general review only)
- 탐지 결과 Hint 패턴 추가: 성공 시 `💡`, 명시적 플래그 + 미탐지 시 `⚠️`, 미설치 + 플래그 없음 시 silent
- `--codex` 모드 에이전트 이름: `codex-review` → `codex-general` (플래그 이름과의 혼동 방지)
- Codex Failure Handling 개선: 인증 실패 시 `⚠️` + `!codex setup` 안내 (기존: 일괄 `ℹ️ unavailable`)
- README에 Codex 통합 셋업 가이드 추가 (설치 → 인증 → 자동 탐지 3단계)

---

### 2026-04-02: 로컬 테스트 워크플로우 개선

**CLAUDE.md**
- 브랜치 테스트 워크플로우 교체: marketplace 전환 방식 → 파일 복사 + `/reload-plugins` 방식
- 기존 marketplace 전환 방식의 `path` 잔여물 버그([#9537](https://github.com/anthropics/claude-code/issues/9537)) 주의 사항 명시

---

### 2026-04-02: /handoff 템플릿 재설계 — 지시형 프롬프트 구조로 전환

**handoff**
- `Context / Delta / Next Action` 고정 3-section 구조 폐지
- 지시문(Directive Line) + 섹션 풀(Section Pool) 조합 구조로 전환
- 지시문: 첫 줄/블록에 다음 세션의 행동을 명시, 스킬 추천 시 copy-paste로 즉시 트리거 가능
- 섹션 풀 6종: `상황`(필수), `원인`, `진행 상황`, `판단 필요 사항`, `조사 결과`, `참고`
- 시나리오별 조합 패턴 5종 제시 (clear task, direction needed, partial progress, investigation complete, skill continuation)
- Confirmation Flow: "thin delta" 등 폐지된 섹션 참조 제거

---

### 2026-04-02: /commit amend 자동 감지 및 --no-amend 플래그 추가

**commit**
- Amend Detection: 직전 커밋과 파일 겹침 감지 시 자동 amend 실행
- Push 상태 인식: 미push 시 amend만, push 완료 시 amend + `--force-with-lease` 자동 실행
- `--no-amend` 플래그: amend 감지 건너뛰고 항상 새 커밋 생성
- Merge commit skip: 직전 커밋이 merge commit이면 amend 감지 건너뜀
- Hint 패턴: 자동 실행 결과를 blockquote 형식으로 사용자에게 고지

**CLAUDE.md**
- 터미널 렌더링 가이드라인에 "Hint 패턴" 섹션 추가 (⚡ 자동 실행, 💡 정보, ⚠️ 주의)
- 로컬 테스트에 amend 시나리오 및 `/commit --no-amend` 테스트 케이스 추가
- "테스트 안내 흐름" 가이드라인 추가

---

### 2026-04-01: /handoff Detection 순서 수정 및 /copy 호환성 개선

**handoff**
- Detection 순서 변경: 대화 맥락 분석(Layer 2)을 최우선으로, Artifact 검색(Layer 4)은 대화 맥락 기반으로 스코핑 — 이전 세션의 무관한 artifact가 handoff에 포함되는 문제 해결
- Handoff Zone을 fenced markdown 코드블록으로 감싸 `/copy` 피커에서 선택 복사 가능 — Meta Zone 없이 handoff 내용만 복사
- Adaptive Section Rules: Context 섹션에서 대화 맥락이 항상 주가 되고, artifact는 보충 참조로 변경

**로컬 테스트 가이드 보완**
- CLAUDE.md: `/reload-plugins` 단계 추가 — SKILL.md 수정 후 캐시 갱신 절차 명시

---

### 2026-04-01: /handoff 스킬 추가 — 세션 이관 handoff 프롬프트 생성

**Handoff 스킬 신규 추가 (handoff)**
- `/handoff` slash command 신규 생성
- Detection Cascade: 아티팩트(.omc/specs/, .omc/plans/) → Git 상태 → 대화 컨텍스트 우선순위 자동 감지
- Reference-not-repeat 원칙: 아티팩트 내용을 반복하지 않고 경로만 참조하여 노이즈 최소화
- Sentinel 기반 스킬 추천: code-review의 Codex zero-cost detection 패턴 재사용 — OMC, Codex, git-claw 스킬 자동 감지 후 다음 세션에 적합한 스킬 추천
- 2-zone 출력 구조: Meta Zone(터미널 전용) + Handoff Zone(/copy 대상) 분리
- 확인 흐름: 아티팩트 존재 시 즉시 출력, 대화 요약 시 사용자 확인 후 출력
- `-y` 플래그: 확인 없이 즉시 출력
- Topic 필터: `/handoff auth 리팩토링`으로 특정 주제에 집중한 handoff 생성
- Read-only 스킬: git 상태 변경 없음, 파일 생성 없음

**문서 업데이트**
- CLAUDE.md: 프로젝트 개요(7개 slash command), 아키텍처 트리, 로컬 테스트, 스킬 공통 규칙에 handoff 반영
- marketplace.json: skills 배열 및 description 업데이트
- README.md, README.ko.md: 스킬 소개, 설치 명령어, CLAUDE.md 연동 섹션 업데이트

### 2026-04-01: `-g`/`--graph` Mermaid 변경 흐름 그래프 기능 추가

**code-review**
- `-g`/`--graph` 플래그를 Mermaid 플로우차트 기능으로 재정의
- PR 모드 전용: GitHub에서 Mermaid 네이티브 렌더링, 터미널 프리뷰에서는 한 줄 요약
- Working Dir/Path 모드에서는 렌더링 대상이 없어 `-g` 무시
- 관계 미감지 시 skip condition 및 사유 고지
- edge 유형: code-level (imports, calls 등) + conceptual (references, shared logic, configures)

**pr**
- `-g`/`--graph` 플래그 신설, Flag Detection 테이블 추가
- Individual PR / Release PR 양쪽 템플릿에 "변경 흐름" Mermaid 섹션 추가
- code-review와 동일한 graph 분석 프로세스 및 skip condition 적용

**CLAUDE.md**
- "스킬 간 공유 로직" 추적 테이블 추가 (graph 분석, label system)

### 2026-04-01: code-review / review-reply 출력 포맷 전면 개선

**CLAUDE.md 터미널 렌더링 가이드라인 보정**
- `---` 실제 동작 문서화 (수평선 → 짧은 세 대시로 보정)
- Unicode box drawing 구분자(`─`, `━`) 항목 추가
- 3단 위계 가이드 신설: `────────────────────` (섹션) / `---` (항목) / 빈 줄 (내부)

**code-review 터미널 출력 개선**
- Findings summary에 severity 아이콘 추가 (`🔴 · 🟡 · 🟢`)
- Severity 헤더: `### < icon Severity (n) >` 포맷으로 전환
- Severity 간 구분: `────────────────────`, finding 간 구분: `---`
- Finding 번호 체계 도입: `C{n}` / `W{n}` / `I{n}` (Terminal + GitHub 공유)

**code-review GitHub 게시 포맷 개선**
- Key Changes: 파일별 테이블 → 개념적 변경 bullet으로 전환
- Contextual Fallback Mapping: 2단계 → 3단계 line resolution (exact → contextual → unmapped)
- Inline Comment: 번호 체계 + contextual match 템플릿 추가

**review-reply 출력 개선**
- 카테고리 간 구분: `---` → `────────────────────` (Unicode thin × 20)
- 항목 간 구분: 빈 줄 → `---`
- 카테고리 헤더에 `< >` 브래킷 + 카운트 추가
- Summary line: 리뷰어 목록 및 카테고리별 카운트 추가
- Reviewer 표기: `Reviewer said:` → `Reviewer ({name}):`

### 2026-04-01: 터미널 출력 템플릿 위계 구분 개선

- review-reply: numbered list → bold paragraph 전환
- code-review: severity 헤더 간 `---` 유지
- CLAUDE.md: bold paragraph 패턴 문서화
- ⚠️ 이후 같은 날짜의 "출력 포맷 전면 개선"에서 위계 체계가 재설계됨 (Unicode 구분자 도입)

### 2026-04-01: 외부 레포 권한 부족 시 graceful fallback 추가

- pr: `git push` 권한 거부 시 `gh repo fork --remote` 자동 실행 후 재시도
- pr: `gh pr create`에서 `--assignee`/`--label` 권한 부족 시 해당 플래그 제거 후 재시도
- issue: `gh issue create`에서 동일한 `--assignee`/`--label` fallback 적용
- 기존 워크플로우(본인 레포)에는 영향 없음 — 실패 시에만 fallback 경로 진입

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
