---
name: code-review
description: >-
  Review code changes using context-aware multi-agent pipeline with severity-based findings.
  TRIGGER when: user asks to review code, analyze PR quality, check for issues, run code review, or audit changes (e.g., "코드 리뷰해줘", "review this PR", "코드 분석해줘", "리뷰 돌려줘").
  DO NOT TRIGGER when: user is replying to review comments (use review-reply), creating PRs, committing, or performing git operations without review intent.
argument-hint: "[PR번호|경로] [-d|--domain security,perf] [-y|--yes] [-g|--graph] [-s|--sub] [-q|--quick] [--wd] [--no-codex|--codex|--codex-general]"
version: "1.7.0"
allowed-tools: Bash(git *), Bash(gh *), Bash(node *), Bash(find *), Read, Grep, Glob, Agent, TeamCreate, TaskCreate, TaskList, TaskUpdate, TaskGet, SendMessage
---

## Parse Arguments

Parse `$ARGUMENTS` to determine the review mode and flags.

### Mode Detection

| Priority | Mode | Condition | Description |
|----------|------|-----------|-------------|
| 1 | **PR Review (explicit)** | `$ARGUMENTS` contains a standalone numeric token (e.g., `42`) or `--pr <number>` | Review a specific PR's diff + commit context |
| 2 | **Working Dir (forced)** | `--wd` flag is set | Force working directory review, even on a PR branch |
| 3 | **Path Review** | `$ARGUMENTS` contains a file/directory path | Review code at the specified path |
| 4 | **PR Review (auto)** | `$ARGUMENTS` is empty → auto-detect (see below) | Auto-detected PR on current branch |
| 5 | **Working Dir** | `$ARGUMENTS` is empty and no PR detected | Review current working directory changes (staged + unstaged) |

### PR Auto-Detection

When `$ARGUMENTS` is empty and `--wd` is not set, attempt to detect an open PR on the current branch:

```
gh pr list --head "$(git branch --show-current)" --author "@me" --state open --json number,title --jq '.[0]'
```

Key behaviors:
- `--author "@me"` ensures only the current user's PRs are matched, skipping bot PRs (dependabot, renovate, etc.)
- If a PR is found → enter **PR Review (auto)** mode with the detected PR number
- If no PR is found (empty result, detached HEAD, or `main`/`master` branch) → fall through to **Working Dir** mode

### Flag Detection

| Flag | Default | Description |
|------|---------|-------------|
| `-y\|--yes\|--force\|-f` | off | Publish without approval |
| `-g\|--graph` | off | Generate a visual change-flow graph for PR review (Mermaid on GitHub, summary in terminal preview). Ignored in Working Dir / Path modes. |
| `-d\|--domain` | auto | Override domain selection (e.g., `-d security,perf`) |
| `-s\|--sub` | off | Use sub-agents instead of team agents for domain analysis |
| `--pr` | — | Explicit PR mode (e.g., `--pr 42`). Use when path arguments contain digits |
| `--wd` | off | Force Working Dir mode, skipping PR auto-detection |
| `-q\|--quick` | off | Quick mode: single-pass analysis without agent spawn. Auto-detected domains capped at 2, Codex disabled, Info findings omitted, graph skipped. Designed for fast iteration during development/testing. |
| `--no-codex` | off | Disable Codex integration entirely (skip Codex detection and execution) |
| `--codex` | off | Run both Codex general review and adversarial review in parallel |
| `--codex-general` | off | Use Codex general review (`codex:review`) only, without adversarial review |

Codex flag precedence: `--no-codex` > `--codex` > `--codex-general` > default (adversarial only).
Only one Codex mode flag should be used at a time. If multiple are present, the highest-precedence flag wins.

If `$ARGUMENTS` contains explicit publish intent ("comment 달아", "바로 올려", "게시해", "post it"), treat as `-y`.

### Quick Mode Implicit Effects

When `--quick` is set, the following flags are implicitly forced regardless of other arguments:

| Implicit Override | Effect |
|-------------------|--------|
| `--no-codex` | Codex always disabled |
| `-g` / `--graph` ignored | Graph generation skipped |

Display hint immediately after flag parsing:

> **⚡ Quick mode** — 단일 패스 분석, Critical/Warning 우선 (없으면 Info fallback)

---

## Step 1: Context Builder

Gather all available context based on the review mode.

### PR Review Mode

```
gh pr view {number} --json title,body,baseRefName,headRefName,commits,files,labels --jq '.'
```

```
gh pr diff {number}
```

```
git log --oneline {baseRefName}..{headRefName} 2>/dev/null | head -30
```

Read the PR description to understand the author's intent. This is critical for cross-validation.

### Working Dir Mode

```
git diff HEAD --stat
```

```
git diff HEAD
```

```
git log --oneline -5
```

If there are no changes (no staged or unstaged diffs), inform the user and stop.

Note: `git diff HEAD` only includes tracked files. Untracked (new) files are not included. If the user wants new files reviewed, they should stage them first (`git add`) before running `/code-review`.

### Path Review Mode

Use Glob to list files under the specified path. Read each file's content.

```
git log --oneline -10 -- {path}
```

### Common (all modes)

For each changed file, read sufficient surrounding context (at least 30 lines around each change hunk) to understand the code's purpose. Use Read to examine files directly.

If `-g` (graph) flag is set **and the mode is PR Review**, build a **change-flow graph** for the GitHub review output. In Working Dir / Path mode, `-g` is ignored — the terminal cannot render Mermaid and there is no GitHub publishing target.

1. **Trace relationships**: For each changed file, identify imports, exports, function calls, and data flow to/from other changed files using Grep.
2. **Map edges**: Record directional relationships — code-level (`imports`, `calls`, `extends`, `emits/consumes`) and conceptual (`references`, `shared logic`, `configures`).
3. **Group by module**: If changed files exceed 7, group by parent directory into logical modules (subgraph nodes). Individual files become child nodes.
4. **Construct Mermaid source**: Build a `flowchart LR` diagram. Nodes = changed files (or module groups). Edges = relationships found in step 2, labeled with relationship type.

**Skip condition**: If no edges (relationships) are found between changed files after step 2, skip graph generation entirely. In the terminal preview, show: `📊 Change Graph: 파일 간 관계가 감지되지 않아 그래프를 생략합니다`. In the GitHub format, omit the graph section silently.

The graph is rendered differently per output format (see Step 5):
- **GitHub**: Full Mermaid code block (native rendering).
- **Terminal (PR preview only)**: One-line summary with module/file counts and primary flow path.

---

## Step 2: Domain Router

Determine which domain agents to activate based on changed files.

### Auto-Activation Rules

| File Pattern | Activated Domains |
|-------------|-------------------|
| `.tsx`, `.jsx`, `.css`, `.html`, `.vue`, `.svelte` | Architecture, Domain Logic |
| `.sql`, `.prisma`, `*migration*` | Performance, Security |
| `auth/`, `middleware/`, `security/`, `*token*` | Security |
| `*.test.*`, `*.spec.*` | Domain Logic |
| `Dockerfile`, `k8s/`, `terraform/`, `*.yml` (CI) | Architecture, Security |
| `package.json`, `requirements.txt`, `go.mod` | Security |
| General source code (`.ts`, `.py`, `.go`, `.java`, `.rs`, etc.) | Architecture, Domain Logic |

Default (no pattern match): Architecture + Domain Logic.

### Override

If `--domain` flag is provided, use ONLY the specified domains regardless of file patterns.

Collect the union of all activated domains from all changed files. Deduplicate.

### Quick Mode Domain Cap

When `--quick` is set (and `--domain` is NOT provided), cap the auto-detected domains to **at most 2** using this priority:

```
Security > Domain Logic > Architecture > Performance
```

Process: run auto-detection as normal, then sort by priority and take the top 2. If auto-detection returns ≤ 2, use all of them as-is.

When `--domain` is explicitly provided alongside `--quick`, respect the override — do not cap.

---

## Step 2.5: Codex Detection

Determine whether the Codex plugin is available and resolve the Codex execution mode.

**Quick mode**: Skip this entire step. Codex mode is already forced to **disabled** (see Quick Mode Implicit Effects). Proceed directly to Step 3.

### Companion Detection

**REQUIRED: Run this command before proceeding to Mode Resolution.**

```bash
COMPANION=$(find ~/.claude/plugins/cache/openai-codex -name "codex-companion.mjs" -print 2>/dev/null | head -1)
```

- Output is non-empty → Codex **available**, store path as COMPANION_PATH
- Output is empty → Codex **unavailable**

**Do not proceed to Mode Resolution until this command has been executed and the result evaluated.**

### Mode Resolution

| Priority | Condition | Codex Mode | Hint |
|----------|-----------|-----------|------|
| 1 | `--no-codex` flag is set | **disabled** | None |
| 2 | Codex unavailable + `--codex` or `--codex-general` flag set | **disabled** | `⚠️ Codex unavailable` — {flag} 요청했으나 companion을 찾을 수 없습니다 |
| 3 | Codex unavailable + no Codex flag | **disabled** | None |
| 4 | `--codex` flag is set | **both** | `💡 Codex detected` — review + adversarial 병렬 실행 |
| 5 | `--codex-general` flag is set | **review** | `💡 Codex detected` — general review 실행 |
| 6 | Default (no Codex flag) | **adversarial** | `💡 Codex detected` — adversarial review 실행 |

Companion subcommands per mode:
- **adversarial**: `adversarial-review --wait`
- **review**: `review --wait`
- **both**: `review --wait` + `adversarial-review --wait`

In PR mode, append `--base {baseRefName}` to each subcommand to scope the review to the PR diff.

Display the resolved Hint (if any) immediately after detection, before launching domain agents. Hints follow the project's Hint 패턴 (`> **{icon} {action}** — {reason}`).

Store the resolved mode for use in Steps 3, 4, and 5. Each spawned Codex agent re-resolves the companion path independently via `find` since agents run in separate contexts. If mode is **disabled**, skip all Codex-related logic in subsequent steps and proceed exactly as before (full backward compatibility).

---

## Step 3: Domain Agents (Parallel Execution)

Launch activated domain agents in parallel. Each agent receives the full diff and context from Step 1.

### Quick Mode: Single-Pass Analysis

When `--quick` is set, skip all agent spawning (team and sub-agent). Instead, perform the analysis directly in the main context:

1. For each activated domain (max 2 from Step 2), analyze the diff using that domain's Investigation Protocol (see Domain Definitions below).
2. Produce findings in the same format as agent results: title, file path, primary line number, occurrence count, description, and action line.
3. Produce findings for **all severity levels** (Critical, Warning, Info). The output step decides which severities to display based on results (see Step 5 Quick Mode Output).
4. Skip Codex execution entirely (mode is **disabled**).

This eliminates the overhead of agent creation, context window allocation, and inter-agent communication. Proceed directly to Step 4 after analysis.

### Execution Mode (Normal)

| Mode | Condition | Description |
|------|-----------|-------------|
| **Team agents** (default) | `-s`/`--sub` flag NOT set | Each domain runs as an independent team agent with its own context window. Better result quality for large diffs. |
| **Sub-agents** | `-s`/`--sub` flag set | Each domain runs as a sub-agent via the Agent tool. Results return to the main context. Lighter for small diffs. |

### Team Agent Mode (default)

Used when `-s`/`--sub` flag is NOT set. Each domain agent gets its own context window.

1. `TeamCreate` — name: `code-review`.
2. For each activated domain:
   - `TaskCreate` — subject: `"{Domain} domain analysis"`. Description: changed files relevant to this domain and the domain-specific prompt/protocol from Domain Definitions.
   - Spawn teammate via `Agent` with `team_name: "code-review"`, `name` set to domain name (lowercase, e.g., `"security"`, `"architecture"`), `model: "opus"`. Prompt: domain-specific prompt from Domain Definitions (including Common Prompt Suffix), full diff from Step 1, changed file context, and finding format (title, file path, primary line number, occurrence count, description, and action line per severity).
   - `TaskUpdate` — set `owner` to the agent name.
3. Monitor `TaskList` until all domain tasks complete. Collect findings from agent messages.
4. Shut down agents via `SendMessage` with `shutdown_request`.
5. Pass aggregated findings to Step 4.

Fallback: if `TeamCreate` is unavailable, switch to Sub-agent mode.

### Sub-agent Mode

Used when `-s`/`--sub` flag IS set, or as fallback. For each activated domain, launch an agent via `Agent` in parallel with `model: "opus"`. Prompt: domain-specific prompt from Domain Definitions (including Common Prompt Suffix), full diff from Step 1, changed file context, and finding format. Results return to the main context.

### Domain Definitions

Each domain agent — whether team or sub — receives its domain-specific prompt via the Agent tool's `prompt` parameter. All agents are spawned with `model: "opus"`. The prompt for each domain consists of: the domain-specific prompt below, followed by the Common Prompt Suffix.

#### Common Prompt Suffix

Append the following sections to every domain agent prompt:

```
## Constraints
- You are read-only. Do not attempt to modify any files.
- Write findings in the language configured in the project's CLAUDE.md. If no language is configured, follow the user's conversational language.
- If no issues are found, return an empty findings list (no items) and state "No issues found." Do not manufacture findings.

## Severity Criteria
- Critical: Security vulnerability, data loss risk, or crash-inducing bug
- Warning: Potential bug, performance issue, or maintainability concern
- Info: Suggestion, minor improvement, or style note

## Output Format
Return findings as a structured list. Each finding must contain:
- title: Short descriptive name (do not repeat the file name)
- file_path: Exact file path
- primary_line: Line number in the NEW version of the file where the issue is most visible; this is the canonical line-reference field for evidence requirements
- occurrence_count: Number of instances of this pattern in the diff
- description: Issue explanation (reference locations by function name or code pattern; do not repeat line numbers here, since `primary_line` provides the line-based evidence)
- action: For Critical/Warning — suggested fix. For Info — one of: Accept (intentional, no action needed) / Monitor (could become an issue at scale) / Won't Fix (known limitation, not worth addressing), with reason.
```

#### 🛡️ Security

```
You are a security engineer conducting a focused security audit of code changes.
You specialize in identifying exploitable vulnerabilities before they reach production.
Prioritize findings by: severity × exploitability × blast radius.

## Investigation Protocol (follow this order)
1. Scan for hardcoded secrets (API keys, passwords, tokens, connection strings)
2. Check input validation: all user inputs sanitized? Parameterized queries?
3. Identify injection vectors: SQL, XSS, command injection, path traversal, SSRF
4. Review authentication/authorization: session management, JWT validation, CSRF protection on state-changing operations, access control on every route
5. Verify cryptographic choices: strong algorithms (AES-256, bcrypt/argon2), proper key management, PII encrypted at rest
6. Audit dependency changes: known CVEs in added/updated packages, lock file integrity
7. Check security configuration: CORS, CSP headers, debug flags, TLS, file upload validation (type/size/content)
8. Assess security logging: auth failures and access denials logged? No sensitive data in logs?

## Evidence Gate
Every finding MUST cite the exact file path and line number. If you cannot point to a specific line in the diff or surrounding context, do not report it.
```

#### ⚡ Performance

```
You are a performance engineer analyzing code changes for runtime efficiency issues.
Focus on issues that degrade under real-world load, not micro-optimizations.

## Investigation Protocol (follow this order)
1. Identify algorithmic complexity: O(n²) or worse in hot paths, unnecessary nested loops
2. Detect database anti-patterns: N+1 queries, missing indexes, unoptimized joins
3. Check async contexts: blocking operations in event loops, missing parallelization of independent I/O
4. Assess memory patterns: unbounded growth, large object retention, missing cleanup
5. Review caching: missing cache for repeated expensive operations, cache invalidation correctness
6. Check resource management: connection pool sizing, file handle leaks, stream backpressure

## Evidence Gate
Every finding MUST cite the exact file path and line number. Only report issues with measurable impact under realistic load. Do not flag theoretical micro-optimizations.
```

#### 🏗️ Architecture

```
You are a staff engineer reviewing code changes for long-term maintainability and design coherence.
You evaluate whether changes align with existing codebase patterns and introduce sustainable design decisions.

## Investigation Protocol (follow this order)
1. Check pattern consistency: does this change follow established patterns in the codebase? Use Grep to find similar implementations and compare approaches.
2. Evaluate SOLID principles: SRP (single reason to change?), DIP (depends on abstractions?), OCP (extensible without modification?)
3. Assess coupling: are new dependencies appropriate? Is the dependency direction correct?
4. Review API contracts: are interfaces/types changed backward-compatibly? Are contracts clear?
5. Check module boundaries: does the change respect existing boundaries? Is responsibility placed correctly?
6. Assess technical debt: does this change introduce shortcuts that will compound?

## Evidence Gate
Every finding MUST cite the exact file path and line number. Reference the existing pattern or file that the change should align with. When recommending a change, state the trade-off (what is gained vs. what is sacrificed). Do not report subjective style preferences.
```

#### 🔍 Domain Logic

```
You are a senior engineer who owns this codebase's business domain, reviewing changes for correctness.
You focus on whether the code does what it's supposed to do, handles all cases, and doesn't break existing behavior.

## Investigation Protocol (follow this order)
0. Verify implementation matches stated intent: does the code solve the problem described in the PR/commit? Anything missing? Anything extra that wasn't requested?
1. Verify business rule correctness: are conditions, thresholds, and control flow correct?
2. Check error handling completeness: all error paths handled? Errors propagate correctly? Resource cleanup?
3. Test edge cases mentally: null/undefined, empty collections, boundary values (0, -1, MAX), concurrent access
4. Identify race conditions: shared state modifications, async ordering assumptions, missing locks/transactions
5. Verify type safety: implicit coercions, unchecked casts, generic type erasure
6. Check state management: initialization order, invalid state transitions, stale data

## Scope
Your scope is correctness and behavioral soundness. Do not flag style, pattern consistency, or maintainability concerns — the Architecture domain covers those.

## Evidence Gate
Every finding MUST cite the exact file path and line number. Describe the specific input or scenario that triggers the bug. Do not report hypothetical issues without a concrete trigger.
```

Each finding must include: title, file path, **primary line number**, occurrence count, description, and action line — `suggested fix` for Critical/Warning severity, `recommendation label` (Accept / Monitor / Won't Fix) for Info severity.

**Primary line number**: The line number in the new version of the file where the issue is most clearly visible. Required for inline comment targeting in PR mode. In the finding's description text, reference locations by section heading, function name, or code pattern — not by raw line number (line numbers are used only for comment placement, not in the output text).

### Codex Parallel Execution

If Codex mode (from Step 2.5) is NOT **disabled**, launch Codex alongside the domain agents in parallel. Codex runs as an independent second reviewer — it is NOT a domain agent, but its execution is concurrent with domain agents.

Codex is invoked via the companion runtime directly through Bash, NOT via `Skill("codex:...")`. The `codex:adversarial-review` and `codex:review` skills have `disable-model-invocation: true`, which blocks `Skill()` invocation. The companion runtime (`codex-companion.mjs`) has no such restriction and is the official programmatic interface for invoking Codex from Claude Code.

#### Team Agent Mode (default, `-s`/`--sub` NOT set)

For each Codex subcommand to invoke (per the mode table in Step 2.5):

1. `TaskCreate` — subject: `"Codex {mode} review"` (e.g., `"Codex adversarial review"`).
2. Spawn teammate via `Agent` with:
   - `team_name: "code-review"`
   - `name: "codex"` (or `"codex-general"` / `"codex-adversarial"` when mode is **both**, to distinguish the two)
   - Prompt: Instruct the agent to run the companion via Bash and return the findings. The Bash command:
     ```
     COMPANION=$(find ~/.claude/plugins/cache/openai-codex -name "codex-companion.mjs" -print 2>/dev/null | head -1) && node "$COMPANION" {subcommand} --wait
     ```
     Where `{subcommand}` is `adversarial-review` or `review` per the mode table. Use `--base {baseRefName}` for PR mode.
3. `TaskUpdate` — set `owner` to the agent name.

The Codex agent(s) run in parallel with domain agents (Security, Architecture, etc.) within the same `code-review` team. They appear in `TaskList` output alongside domain agents, satisfying traceability (Acceptance Criterion F).

#### Sub-agent Mode (`-s`/`--sub` set)

For each Codex subcommand to invoke:

- Launch via `Agent` with `name: "codex"` (or `"codex-general"` / `"codex-adversarial"` for **both** mode). No `team_name`. The agent runs the companion via Bash (same command as Team Agent Mode) and returns findings.

Launch Codex agents at the same time as domain agents — do NOT wait for domain agents to finish first.

### Codex Failure Handling

If a Codex agent reports a non-zero exit code or returns an error (e.g., quota exhausted, authentication failure, network error):

1. **Do NOT retry** — treat the Codex contribution as unavailable for this run.
2. **Findings = empty** — proceed with domain agent findings only. Do not attempt to parse error output as findings.
3. **Terminal notice** — classify the error and display the appropriate message:

| Error Signal | Terminal Notice |
|-------------|----------------|
| stderr contains `auth`, `login`, `API key`, `unauthorized`, `401` | `⚠️ Codex auth required` — `!codex setup` 실행 권장 |
| Any other non-zero exit | `ℹ️ Codex: unavailable (skipped)` |

4. **GitHub format** — do NOT include any Codex failure notice. Codex availability is an internal infrastructure detail, not relevant to PR reviewers.

### Fallback (non-Claude Code runners)

If neither team agents nor sub-agents are available (e.g., Codex CLI, Gemini CLI as the runner platform), perform all domain analyses sequentially in a single pass. Analyze each domain's Investigation Protocol one by one and collect findings.

---

## Step 4: Cross-Validation

This is the quality gate. Review ALL findings from domain agents against the full context to filter false positives.

### Quick Mode: Lightweight Validation

When `--quick` is set, perform a reduced validation pass:

1. **Expanded context**: Read at least 15 lines around the flagged location using Read (half of normal).
2. **Sanity check**: Verify the finding references real code (not a false match from diff noise).

Skip git history, comments/docs search, and PR description cross-reference. Apply Confirmed / Dismissed verdicts only (no Demoted). This trades thoroughness for speed — obvious false positives are still caught, but edge cases may pass through.

### Normal Mode

For each finding:

1. **Expanded context**: Read at least 30 lines around the flagged location using Read.
2. **Git history**: Check if the code was intentionally written this way:
   ```
   git log --oneline -5 -- {file}
   ```
3. **Comments/docs**: Search for TODO, FIXME, or design notes near the flagged code. Check if the project's CLAUDE.md or other documentation addresses the pattern.
4. **PR description/commit messages**: Cross-reference against the author's stated intent (from Step 1).

### Verdict per finding

| Verdict | Action |
|---------|--------|
| **Confirmed** | Real issue — include in final output |
| **Demoted** | Real but already acknowledged — downgrade to Info with context note |
| **Dismissed** | False positive — remove from output |

Log dismissed findings internally (do not output them) to avoid noise.

### Codex Findings Integration

If Codex mode is NOT **disabled**, collect findings from the Codex agent(s) after they complete. Codex findings join the cross-validation process identically to domain agent findings:

1. **Merge into unified pool**: Combine Codex findings with domain agent findings before cross-validation.
2. **Apply the same verdict process**: Each Codex finding undergoes the same Confirmed / Demoted / Dismissed evaluation (expanded context, git history, comments/docs, PR description).
3. **Tag preservation**: Preserve the Codex origin tag on each finding (see Step 5 for tag rules). The tag indicates which Codex subcommand produced the finding.

Codex findings are NOT given special treatment — they must pass the same quality gate as domain agent findings.

### Deduplication

When multiple agents (domain or Codex) flag the same code location:
- **Same root cause**: Merge into a single finding. Keep the higher severity and credit all relevant sources (e.g., `Architecture • Domain Logic`, `Architecture • Codex`). Codex tags follow Step 5 Source Tags rules (`Codex` or `Codex Adv`).
- **Different concerns**: Keep as separate findings, each under its own domain.

---

## Step 5: Output Generator

Produce severity-first structured output.

### Quick Mode Output

When `--quick` is set, the output severity is determined by results:

- **Critical/Warning 1건 이상**: Critical/Warning만 출력, Info 생략. Summary: `Findings: 🔴 {n} critical · 🟡 {n} warnings`.
- **Critical/Warning 0건, Info 1건 이상**: Info findings를 fallback으로 출력 (Recommendation labels 포함). Summary: `Findings: 🟢 {n} info`.
- **전체 0건**: Terminal/GitHub 공통 zero-findings 규칙 적용 (`✅ No issues found.`).

Common rules:
- **Source Tags** — same as Normal mode, using domain names.
- **Graph** — always omitted (see Quick Mode Implicit Effects).
- **Codex tags** — not applicable (Codex disabled).
- **GitHub format**: If publishing in PR mode, use the same GitHub format with the same severity filtering rules above.

### Severity Levels

| Severity | Icon | Criteria |
|----------|------|----------|
| Critical | 🔴 | Security vulnerability, data loss risk, crash-inducing bug |
| Warning | 🟡 | Potential bug, performance issue, maintainability concern |
| Info | 🟢 | Suggestion, minor improvement, style note |

### Recommendation Labels

Info findings use a recommendation label instead of a fix suggestion. The label conveys the reviewer's assessment of whether action is needed.

| Label | Meaning | When to use |
|-------|---------|-------------|
| **Accept** | Acknowledged, no action needed | Intentional design choice, acceptable trade-off, or cosmetic preference |
| **Monitor** | No immediate action, track over time | Could become an issue at scale, under load, or after future changes |
| **Won't Fix** | Known limitation, not worth addressing | Cost outweighs benefit, outside scope, or constrained by external factors |

### Source Tags

Each finding displays a source tag after the title (e.g., `**Finding title** — Security`). Domain agent findings use their domain name. Codex findings use tags based on the active Codex mode:

| Codex Mode | Source(s) | Tag(s) |
|-----------|-----------|--------|
| **disabled** | Domain agents only | Domain name (e.g., `Security`, `Architecture`) |
| **adversarial** (default) | Domain agents + Codex adversarial | Domain name / `Codex` |
| **review** (`--codex-general`) | Domain agents + Codex review | Domain name / `Codex` |
| **both** (`--codex`) | Domain agents + Codex review + Codex adversarial | Domain name / `Codex` (review findings) / `Codex Adv` (adversarial findings) |

When Codex mode is **adversarial** or **review** (single source), all Codex findings are tagged `— Codex`.
When Codex mode is **both** (dual source), review findings are tagged `— Codex` and adversarial findings are tagged `— Codex Adv` to distinguish the two sources.

All findings (domain + Codex) are sorted together by severity (Critical → Warning → Info), not grouped by source.

### Output Templates

Write the review in the language configured in the project's CLAUDE.md.
If no language is configured, follow the user's conversational language.
Examples below are in Korean.

There are two output formats depending on the rendering medium:
- **Terminal format**: Used for Working Dir / Path mode, and for PR mode preview (before approval).
- **GitHub format**: Used only for the final PR comment published to GitHub (after approval).

#### Terminal Format

Optimized for CLI readability. No HTML tags, no tables, flat structure.
Three-level visual hierarchy: `────────────────────` (Unicode thin × 20) between severity sections, `---` between findings and after severity headers, blank lines within findings.
Finding title comes first (renders as bold/bright in terminal), file path second.

```markdown
## Code Review: {target}

Domains: {activated domains joined by " • "}{if Codex enabled: " · Codex 🤖"}
Findings: 🔴 {critical_count} critical · 🟡 {warning_count} warnings · 🟢 {info_count} info
{if -g flag set AND PR mode AND relationships found:}
📊 Change Graph: GitHub 게시 시 Mermaid 플로우차트 포함
   {module_count} modules · {file_count} files · 주요 흐름: {primary_flow_path}
{else if -g flag set AND PR mode AND no relationships found:}
📊 Change Graph: 파일 간 관계가 감지되지 않아 그래프를 생략합니다
{end if}

────────────────────

### < 🔴 Critical ({n}) >

---

**C1. {finding title}** — {domain}
`{file}` ({N}곳)

{description}

> **Fix**: {suggestion}

---

**C2. {finding title}** — {domain}
`{file}` ({N}곳)

{description}

> **Fix**: {suggestion}

────────────────────

### < 🟡 Warning ({n}) >

---

**W1. {finding title}** — {domain}
`{file}` ({N}곳)

{description}

> **Fix**: {suggestion}

---

**W2. {finding title}** — {domain}
`{file}` ({N}곳)

{description}

> **Fix**: {suggestion}

────────────────────

### < 🟢 Info ({n}) >

---

**I1. {finding title}** — {domain}
`{file}` ({N}곳)

{description}

> **Recommendation**: {Accept | Monitor | Won't Fix} — {reason}

---

**I2. {finding title}** — {domain}
`{file}` ({N}곳)

{description}

> **Recommendation**: {Accept | Monitor | Won't Fix} — {reason}
```

#### GitHub Format: Review Summary

Posted as the pull request review `body`. Contains the severity counts, key changes, and any findings that could not be mapped to diff lines (unmapped findings).

```markdown
## Code Review: {target}

**Domains**: {activated domains joined by " • "}{if Codex enabled: " · **Codex 🤖**"}
**Findings**: {severity counts joined by " · ", e.g., "🔴 1 Critical · 🟡 2 Warning · 🟢 1 Info"}{if unmapped > 0: " ({mapped count} inline, {unmapped count} general)"}

### Summary

{1-2 sentence overview of the PR's purpose and the review's key judgment — e.g., "Codex 통합을 위한 Step 2.5 추가 및 companion runtime 연동. 전반적으로 하위호환성이 잘 유지되나, 스킬 가용성 검증에 보완이 필요하다."}

### Key Changes
- {개념적 변경 1: 무엇을 왜}
- {개념적 변경 2}
- {개념적 변경 3}

{if -g flag set AND relationships found:}

### Change Graph

```mermaid
flowchart LR
  subgraph {module_name}[{Module Display Name}]
    {file_node_id}[{filename}]
  end
  {source_node} -->|{relationship_label}| {target_node}
```

{end if}

{if unmapped findings exist:}

---

### 📋 General Findings

{for each unmapped finding, sorted by severity:}

{severity_icon} **{finding title}** — {domain}
`{file}` ({N}곳)

{description}

{if severity is Critical or Warning:}
> **Fix**: {suggestion}
{else (Info):}
> **Recommendation**: {Accept | Monitor | Won't Fix} — {reason}
{end if}

---

{end for}

Generated with [Claude Code](https://claude.com/claude-code)
```

- **Summary**: 1-2문장으로 PR 목적과 리뷰 핵심 판단을 기술. PR description의 요약이 아닌 리뷰어 관점의 평가.
- **Key Changes**: 개념적 변경사항 bullet. 각 bullet은 하나의 논리적 변경 단위를 기술. 개수 제한 없이, PR이 수행한 변경만큼 기재.
- **Findings count**: unmapped가 0이면 total count만 표시. unmapped가 1 이상이면 `(N inline, M general)` breakdown 추가.

If there are zero findings overall, post: `## Code Review: {target}\n\n✅ No issues found.\n\nGenerated with [Claude Code](https://claude.com/claude-code)`

#### GitHub Format: Inline Comment

Posted as individual review comments, each attached to a specific file and line in the diff. One finding per comment.

````markdown
{severity_icon} **{prefix}{n}. {finding title}** — {domain}

{description}

{if severity is Critical or Warning:}
{if concrete code change:}
```suggestion
{suggested code — only the replacement lines, matching GitHub suggestion block format}
```
{else:}
> **Fix**: {suggestion}
{end if}
{else (Info):}
> **Recommendation**: {Accept | Monitor | Won't Fix} — {reason}
{end if}
````

Numbering uses the same scheme as Terminal format: `C{n}` / `W{n}` / `I{n}`, starting from 1 per severity. Terminal preview and GitHub inline comments share the same numbers for cross-reference.

For contextual match findings (mapped via contextual fallback):

````markdown
{severity_icon} **{prefix}{n}. {finding title}** — {domain}

📍 이 변경의 영향: `{affected_file}` ({N}곳)

{description}

{if severity is Critical or Warning:}
{if concrete code change:}
```suggestion
{suggested code — only the replacement lines, matching GitHub suggestion block format}
```
{else:}
> **Fix**: {suggestion}
{end if}
{else (Info):}
> **Recommendation**: {Accept | Monitor | Won't Fix} — {reason}
{end if}
````

Use GitHub `suggestion` blocks when the fix is a concrete, localized code change (variable rename, parameter addition, line replacement). The suggestion block enables one-click "Apply suggestion" in GitHub's UI. Use `> **Fix**: ...` for directional or structural suggestions that span multiple locations. Info findings always use `> **Recommendation**: ...` — suggestion blocks are not applicable.

### Formatting Rules

**Common rules (both formats)**:
- Location: file path on its own line, occurrence count only (e.g., "(3곳)", "(1곳)"). No line numbers (`L42`, `L58` 등) anywhere — descriptions and Fix suggestions reference locations by section heading, function name, or searchable code pattern instead.
- Finding title must NOT repeat the file name (location is already on its own line).
- Omit severity sections that have 0 findings.
- **Bullet management**: If a single finding has more than 5 sub-points, consolidate.
- **Commit SHA references**: Never use backticks around SHAs. Use plain text or markdown links.

**Graph rules (when `-g` flag is set)**:
- Mermaid direction: `flowchart LR` (left-to-right) for readability.
- Node labels: filename only (no full path). Use `[filename.ext]` format (basename + extension).
- Edge labels: relationship type — code-level (`imports`, `calls`, `extends`, `emits/consumes`, `reads/writes`) or conceptual (`references`, `shared logic`, `configures`).
- Module grouping: When changed files > 7, group by parent directory using `subgraph`. When ≤ 7, show individual file nodes without subgraph.
- Terminal: One-line summary only — module count, file count, primary flow path (longest chain of connected nodes, max 4 nodes joined by ` → `).
- GitHub: Full Mermaid code block placed after Key Changes, before General Findings.

**Terminal-specific rules**:
- No `<details>` or HTML tags — they don't render in CLI.
- Summary line with severity icons: `Findings: 🔴 1 critical · 🟡 3 warnings · 🟢 1 info`.
- Severity headers use `### < {icon} {Severity} ({n}) >` format with `< >` brackets.
- Severity sections are separated by `────────────────────` (Unicode thin box drawing × 20).
- Severity header is followed by `---` before the first finding. Findings within the same severity are also separated by `---`.
- Each finding is numbered with a severity prefix: `C{n}` (Critical), `W{n}` (Warning), `I{n}` (Info), starting from 1 per severity.
- Finding title first (bold — renders bright in CLI), file path second.
- Action line by severity: Critical/Warning use `> **Fix**: ...` (natural language, referencing by section/pattern). Info uses `> **Recommendation**: {Accept | Monitor | Won't Fix} — {reason}` to convey whether action is needed.
- Domains with no findings: omit entirely (no "✅ ... No issues found" line).
- Zero findings overall: display `## Code Review: {target}\n\n✅ No issues found.` — severity sections, summary line 모두 생략.
- Codex failure notice (if applicable): append after the summary line per the error classification in Step 3 Codex Failure Handling (`⚠️` for auth errors, `ℹ️` for other failures). Only shown when Codex mode was NOT disabled but companion returned non-zero exit. Do NOT include in GitHub format.

**GitHub-specific rules (inline review comments)**:
- Review Summary: severity counts and key changes at the top. Unmapped findings (after contextual mapping) included below under "General Findings" as fallback only.
- Inline Comment: one finding per comment. No `<details>` tags — each comment is self-contained.
- Action line format in inline comments:
  - Critical/Warning with concrete, localized code change → GitHub `suggestion` block (enables one-click apply).
  - Critical/Warning with directional or multi-location suggestion → `> **Fix**: ...` natural language.
  - Info → `> **Recommendation**: {Accept | Monitor | Won't Fix} — {reason}` (no suggestion block).
- Domains with no findings: omit entirely from both summary and inline comments.

---

## Step 6: Publisher

Based on the mode and flags, publish the review output.

**Quick mode**: No changes to publishing behavior. The same Working Dir / PR mode flow applies.

### Working Dir / Path Mode

Display the review output using **Terminal format** directly to the user. No GitHub publishing.

### PR Review Mode

PR mode uses a two-phase flow: preview in terminal, then publish to GitHub.

#### Phase 1: Preview (Terminal format)

Generate the review using **Terminal format** and display it to the user in the conversation.

| Condition | Next step |
|-----------|-----------|
| `-y` or `-f` flag present | Skip preview, go directly to Phase 2 |
| `$ARGUMENTS` contains publish intent ("comment 달아", "바로 올려", "게시해", "post it") | Skip preview, go directly to Phase 2 |
| **Default** | **STOP here. Show the Terminal format output and ask: "PR에 게시할까요?" Do NOT proceed until the user approves.** |

#### Phase 2: Publish (Inline Review)

After approval (or auto-publish flag), publish findings as **inline review comments** via the GitHub Pull Request Review API. Each finding becomes an individual comment attached to the relevant file and line in the diff, enabling per-finding resolution and reply through GitHub's native UI.

**1. Resolve line targets**

For each confirmed finding, verify its primary line number exists in the PR diff:

```
gh pr diff {number}
```

For each confirmed finding, resolve its target line in the PR diff using a three-tier strategy:

1. **Exact match**: The finding's file + line appears in a diff hunk (added line `+`,
   or context line on the RIGHT side). Include as an inline comment at that location.

2. **Contextual match**: Exact match failed. Search the diff for the root cause change
   that triggered this finding:
   - Rename not propagated → map to the diff line where the rename occurred
   - Missing entry that should accompany new entries → map to the nearest new entry line
   - Dead code from a removal → map to a nearby RIGHT-side context or addition line adjacent to the removal
   Include as an inline comment at the contextual location (must be a RIGHT-side line). Prepend to the comment body:
   `📍 이 변경의 영향: \`{affected_file}\` ({N}곳)`
   followed by the original finding description.

3. **Unmapped**: Neither exact nor contextual match is possible (e.g., finding references
   a file/pattern with no related changes in the diff). Include in the review body under
   "📋 General Findings" as a last resort.

**2. Build review payload**

Construct a JSON payload for the Review API:

```json
{
  "commit_id": "{head_sha}",
  "event": "COMMENT",
  "body": "{Review Summary template}",
  "comments": [
    {
      "path": "{file}",
      "line": {end_line_number},
      "side": "RIGHT",
      "start_line": {start_line_number},
      "start_side": "RIGHT",
      "body": "{Inline Comment template}"
    }
  ]
}
```

- `commit_id`: Obtain from `gh pr view {number} --json headRefOid --jq '.headRefOid'`
- `body`: Review Summary template (severity counts + key changes + unmapped findings if any + footer)
- `comments`: Array of mapped findings, each using the Inline Comment template
- For single-line findings, omit `start_line` and `start_side` — only `line` and `side` are needed.
- Serialize the JSON payload using `jq -n` or write to a temp file — do NOT manually escape strings in a heredoc. Comment bodies contain multi-line markdown, code fences, and `suggestion` blocks that break raw string interpolation.

If there are no mapped findings (all unmapped), the `comments` array is empty. The review body contains all findings.

**3. Submit**

```
jq -n \
  --arg commit_id "$HEAD_SHA" \
  --arg body "$REVIEW_BODY" \
  --argjson comments "$COMMENTS_JSON" \
  '{commit_id: $commit_id, event: "COMMENT", body: $body, comments: $comments}' \
| gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input -
```

Where `{owner}/{repo}` is obtained from `gh repo view --json nameWithOwner --jq '.nameWithOwner'`.

**4. Fallback**

If the Review API call fails (e.g., 422 due to invalid line mapping):
1. Move all inline comments to the Review Summary body as General Findings, then resubmit with an empty `comments` array. GitHub's 422 does not specify which comment failed, so individual retry is not possible.
2. If the retry also fails or the API is unavailable, fall back to posting the full review as a single PR comment:
   ```
   gh pr comment {number} --body "$(cat <<'EOF'
   {Review Summary with ALL findings included in body}
   EOF
   )"
   ```

---

## Task

1. Parse `$ARGUMENTS` to determine mode (PR / Working Dir / Path) and flags (including Codex flags and `--quick`).
2. **Context Builder**: Gather diff, commit history, related files, and PR description (if applicable).
3. **Domain Router**: Analyze changed file types and activate relevant domains. Respect `--domain` override. If `--quick`, cap to 2 domains by priority.
4. **Codex Detection**: If `--quick`, skip (Codex disabled). Otherwise, resolve companion path via `find`, and determine Codex mode (adversarial / review / both / disabled).
5. **Domain Agents + Codex**: If `--quick`, perform single-pass analysis in main context (no agent spawn, all severities). Otherwise, launch activated domain agents in parallel. If Codex is enabled, launch Codex agent(s) via companion runtime concurrently. Collect all findings. If Codex fails (non-zero exit), proceed with domain findings only.
6. **Cross-Validation**: If `--quick`, lightweight validation (context check + sanity only). Otherwise, verify each finding (domain + Codex) against expanded context, git history, comments, and PR intent. Classify as Confirmed / Demoted / Dismissed.
7. **Output Generator**: Produce severity-first structured output with source tags. If `--quick`, show Critical/Warning only; if none, fallback to Info; graph always omitted.
8. **Publisher**:
   - **Working Dir / Path mode**: Display Terminal format output directly. Done.
   - **PR mode without `-y`/`-f`**: Show Terminal format preview → ask "PR에 게시할까요?" → resolve line targets → build inline review payload → publish via Review API ONLY if approved.
   - **PR mode with `-y`/`-f`**: Resolve line targets → build inline review payload → publish via Review API immediately.

**Important:**
- Do NOT publish review comments to GitHub without explicit user approval. The only exceptions are the `-y`/`-f` flag or explicit publish intent in `$ARGUMENTS`. A PR number alone is NOT publish intent.
- Do NOT replace linter checks — focus on semantic, architectural, and logic issues.
- Do NOT suggest auto-fix or auto-merge — this is review only.
- This skill is independent from `review-reply`. `code-review` generates reviews (proactive); `review-reply` responds to received reviews (reactive).
- **Assignee**: If creating GitHub PRs or issues, always include `--assignee @me`.
- **Commit references**: Never wrap commit SHAs in backticks. Use plain text or explicit markdown links.
- Adapt output language to the project's CLAUDE.md language setting.
- When Agent tool is unavailable, perform all analyses sequentially as a single-pass fallback.
