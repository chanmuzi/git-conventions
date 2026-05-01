---
name: commit
description: >-
  Create a git commit following project conventions.
  TRIGGER when: user asks to commit, create a commit, stage and commit, save changes, or any workflow that includes committing (e.g., "commit and push", "commit and create PR", "커밋해줘", "커밋하고 푸시").
  DO NOT TRIGGER when: user is only checking git status/diff without intent to commit.
version: "1.5.0"
allowed-tools: Bash(git *), Read, Grep, Glob
---

## Gather Context

If the current changes are already known from the conversation context (e.g., you just edited the files), skip these commands.
Otherwise, run them to understand the current state:
1. `git status` — check current status and untracked files
2. `git diff HEAD` — review all changes

## Flag Detection

| Flag | Effect |
|------|--------|
| `--amend` | Amend the last commit instead of creating a new one |

Parse `$ARGUMENTS` for flags before proceeding. Natural language equivalents: "amend해줘", "직전 커밋에 합쳐줘", "마지막 커밋 수정" → `--amend`.

## Amend (opt-in only)

> Skip this section if any of the following is true:
> - `--amend` is NOT set
> - The last commit is a merge commit (`git rev-parse --verify HEAD^2 2>/dev/null` succeeds)

When `--amend` is set, amend the last commit with the current changes.

**Step 1 — Check push status:**

```bash
git log @{u}..HEAD --oneline 2>/dev/null
# - Command fails (no upstream): NOT pushed
# - Non-empty output: NOT pushed (unpushed commits exist)
# - Empty output: already pushed → force push required after amend
```

**Step 2 — Execute amend:**

- Stage the relevant files and run `git commit --amend`.
  - Use `--no-edit` if the change is a minor fix within the same intent.
  - Update the commit message if the scope or meaning has changed.

- **NOT pushed** → amend directly and display:

> **⚡ Amend** — `--amend` flag detected

- **Already pushed** → display a warning and ask for confirmation before proceeding:

> **⚠️ Amend + force push** — the last commit has already been pushed. Proceed?

  - If confirmed: amend and run `git push --force-with-lease`, then display:

> **⚡ Force pushed** (`--force-with-lease`)

  - If declined: fall through to Intent Classification (new commit).

**After amend:**
- If no unstaged or untracked changes remain → done. Do not proceed further.
- If any changes remain → proceed to Intent Classification for the remaining changes.

## Intent Classification

> Run this BEFORE drafting any commit message. The output of this step is the source of truth for the commit plan — it cannot be inferred silently in your head.

### Step 1 — Enumerate every changed file

List every changed file (staged + unstaged + untracked) using `git status --porcelain`. Do not summarize; every path must appear in the next step.

### Step 2 — Map each file to exactly one Intent Category

Output the mapping as a markdown list. **You MUST emit this list explicitly, even when there is only one file or only one category.** This is non-optional and non-conditional. Skipping it is the most common cause of misgrouping.

Example output:

```
- web/dashboard.tsx              → app-runtime
- web/AGENTS.md                  → agent-meta
- web/CLAUDE.md                  → agent-meta
- docs/ops/slurm-completing.md   → docs
```

Use the file's role in the project, not its filename or directory alone, to assign a category.

**Categories:**

- `infra-deploy`   — deployment configs, environment templates, ops/setup/teardown/health/seed scripts
- `agent-meta`     — agent instructions, hooks, permission/secret guards (AGENTS.md, CLAUDE.md, .claude/, .codex/, secret-related .gitignore patterns)
- `app-runtime`    — production code implementing one feature or fix
- `build-tooling`  — build/lint/format/type-check configs, dependency manifests
- `docs`           — documentation-only changes (README, docs/, ops runbooks, inline doc-only edits)
- `test`           — test-only additions or updates

If a file genuinely fits two roles, pick the category matching its primary effect at runtime, and note the ambiguity inline (e.g., `web/CLAUDE.md → agent-meta (also touches docs)`).

### Step 3 — Produce the Commit Plan

Build the **Commit Plan** — the canonical, ordered list of commit units this invocation will create. Every downstream guard (Pre-commit invariant, Post-commit self-check) compares against this plan, not against the raw category count.

Construction (apply in order):

1. Start from the file → category mapping in Step 2. The initial baseline is **one commit unit per distinct category**.
2. **Apply Same-intent exceptions** (closed whitelist below). Each matched pattern merges its participating categories into one unit, reducing unit count.
3. **Apply Tie-breaking within a category** (rules below). Each split increases unit count for that category.

The Commit Plan is a list of units, each carrying (a) its files, (b) its source category(ies), and (c) a draft message. The number of commits this invocation creates MUST equal the number of units in the plan.

Push status (already pushed or not) MUST NOT influence the plan. If force push becomes necessary after history rewriting, follow the Amend section's confirmation pattern (display the warning, ask the user, then `git push --force-with-lease`).

Display the plan:

```
**Commit Plan**
1. {type}({scope}): {subject} — N files [category]
   - file1, file2, ...
2. {type}({scope}): {subject} — M files [category]
   - file3, file4, ...
```

### Same-intent exceptions (closed whitelist)

Merging files from different categories into one commit is allowed ONLY when one of these exact patterns applies:

1. Schema/model change + production code that reads or writes the new field.
2. Function signature change + every required call-site update.
3. Production code change + the tests validating exactly that change (`app-runtime` ⊕ `test`).

If your justification is not literally on this list, **split**.

### Anti-patterns (NOT exceptions — these mean SPLIT)

When you catch yourself reasoning along any of these lines, treat it as positive evidence to split, not to merge. These are the most common failure modes of this skill:

- "They tell one story / share a theme / belong together conceptually."
- "The code change and the documentation describing the principle go hand in hand."
- "Operations notes alongside the code that triggered them."
- "Same feature work session / atomic from the user's point of view."
- "Force push is needed anyway, so may as well keep it tight."
- "Reverting them separately would feel weird."
- "The PR description will explain the connection."
- "It's only a few files."

None of these override the category split rule.

### Tie-breaking within one category

When all files share a category but the diff is still large, use these checks to decide further splits:

- Reverting this unit alone removes only one intended change.
- Cherry-picking this unit alone leaves the codebase building, running, and tested.
- One reason drives this unit, not a mix of motivations.
- Splitting would make root-cause isolation easier on future regressions.

Do not split purely along layer boundaries (frontend vs. backend) when both are required for one valid end-to-end change. Do split when each layer is independently verifiable, revertible, and cherry-pickable.

### Worked examples (use as test cases when editing this skill)

These cases show that **commit count can differ from raw category count** depending on exceptions and tie-breaking. Always derive guards and self-checks from the Commit Plan, never from category count alone.

**Example A — straightforward split**

Files: `src/api.ts` (app-runtime), `README.md` (docs).
Categories: 2. Exceptions: none. Tie-breaking: none.
Commit Plan: 2 units → 2 commits.

**Example B — Same-intent exception merges categories (commits < categories)**

Files: `src/auth.ts` (app-runtime), `tests/auth.test.ts` (test).
Categories: 2. Exception #3 (production code + validating tests) applies.
Commit Plan: 1 unit (app-runtime ⊕ test) → 1 commit.

**Example C — Tie-breaking expands one category (commits > categories)**

Files: `src/feature_x.ts`, `src/feature_y.ts` — both `app-runtime`, unrelated motivations.
Categories: 1. Tie-breaking: 2 independently revertible units.
Commit Plan: 2 units → 2 commits.

**Example D — combined**

Files: `src/auth.ts`, `tests/auth.test.ts`, `src/feature_x.ts`, `README.md`.
Categories: 3 (app-runtime, test, docs). Exception #3 merges (`auth.ts` ⊕ `auth.test.ts`); `feature_x.ts` and `README.md` stay separate.
Commit Plan: 3 units → 3 commits (categories=3, but composition differs).

## Commit Message Convention

Format: `{type}: {description}` or `{type}({scope}): {description}`

### Type Prefixes (always lowercase)

| Type | Purpose |
|------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code restructuring without behavior change |
| `style` | Formatting, linting (no logic change) |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `perf` | Performance improvement |
| `chore` | Build, CI, dependencies, config |
| `hotfix` | Urgent production fix |

### Scope (optional)

Add scope in parentheses when the change targets a specific module or component:
- `feat(source-map): ...`
- `fix(full-text): ...`
- `test(query-rewriting): ...`

### Description Rules

- Write the description in the language configured in the project's CLAUDE.md. If no language is configured, follow the user's conversational language.
- Keep the subject line concise — aim for under 50 characters.
- Use natural, fluent phrasing. Do not force-translate well-known technical terms. For example, write `source_map` as-is rather than translating it.
- Focus on WHAT changed and WHY, not HOW.

### Multi-line Body (for complex changes)

When the diff involves multiple files or logical units, add a body after a blank line to explain details:

```
{type}: {subject line}

- {detail 1}
- {detail 2}
- {detail 3}
```

**When to use multi-line:**
- 3+ files changed across different concerns
- Non-obvious reasoning behind the change
- Breaking changes or migration notes

**When single-line is enough:**
- Simple, self-explanatory changes (typo fix, single-file edit, etc.)

### Examples

**Single-line:**
```
feat: 멀티턴 컨텍스트 유지 기능 추가
fix: 요청 DTO 모델 기본값을 settings 기반으로 통일
refactor: 미사용 openai_api_key/openai_base_url 프로퍼티 제거
style: black 포맷팅 적용
perf: Classification max_tokens 30 → 15로 조정
chore: 보안 취약점 패키지 업데이트 (urllib3, langchain-core)
docs: README 환경변수 가이드 업데이트
test: 멀티턴 컨텍스트 유지 기능 테스트 추가
feat(source-map): 검색하지 않은 턴에서도 source_map 지속 반환
fix(full-text): 전문 조회 실패 시 state 누출 방지
hotfix: uv lock synced
```

**Multi-line:**
```
refactor: Gather Context 불필요한 명령 및 중복 step 제거

- commit: git branch, git log 명령 삭제 (gitStatus/컨벤션 정의로 대체)
- pr: git remote show origin, gh pr list 네트워크 호출 제거
- review: Gather Context 섹션 삭제, 카테고리 불일치 수정
```

## Task

> If Amend already committed some or all files, only process the remaining uncommitted changes below. If nothing remains, skip this section.

For each commit unit produced by Intent Classification:

1. **Stage**: add only the files mapped to this unit. Stage them by name; never use `git add -A` or `git add .`.

2. **Pre-commit invariant** — verify staged files match the current commit unit in the Commit Plan:

   ```bash
   git diff --staged --name-only
   ```

   The staged set MUST equal the file list of the unit being processed. Extra or missing files mean re-stage (`git restore --staged <wrong-files>` and re-add). For units formed by a Same-intent exception, the staged set may legitimately span multiple categories — the invariant is plan-unit equality, not category singularity. Do not commit until the staged set matches the current unit exactly.

3. **Compose the message** per the Convention section above. Use a multi-line body when 3+ files are involved or the reasoning is non-obvious.

4. **Commit**. Approval follows the session's tool permission settings.

After all units are committed:

5. **Self-check**: confirm the number of new commits equals the number of units in the Commit Plan from Step 3. Use the final plan after Same-intent exceptions and tie-breaking — never compare against the raw category count, which can over- or under-state the expected commit count. If they don't match, report the mismatch explicitly and stop — do not silently continue to subsequent steps such as PR creation. Recovery options:
   - If unpushed: `git reset --soft HEAD~N` and re-run from Intent Classification.
   - If pushed: warn the user and ask for confirmation before history rewriting + `git push --force-with-lease`, mirroring the Amend section's confirmation pattern.

**Important:**
- Never use `git add -A` or `git add .` — stage specific files by name.
- Never include files that may contain secrets (`.env`, credentials, tokens, etc.).
- When multiple commit units were planned, do NOT stop after the first commit — handle all units within this single skill invocation.
