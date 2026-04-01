---
name: code-review
description: >-
  Review code changes using context-aware multi-agent pipeline with severity-based findings.
  TRIGGER when: user asks to review code, analyze PR quality, check for issues, run code review, or audit changes (e.g., "코드 리뷰해줘", "review this PR", "코드 분석해줘", "리뷰 돌려줘").
  DO NOT TRIGGER when: user is replying to review comments (use review-reply), creating PRs, committing, or performing git operations without review intent.
argument-hint: "[PR번호|경로] [-d|--domain security,perf] [-y|--yes] [-g|--graph] [-s|--sub] [--wd] [--no-codex|--codex-review|--codex-both]"
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
| `-g\|--graph` | off | Enable code graph analysis (dependency map) |
| `-d\|--domain` | auto | Override domain selection (e.g., `-d security,perf`) |
| `-s\|--sub` | off | Use sub-agents instead of team agents for domain analysis |
| `--pr` | — | Explicit PR mode (e.g., `--pr 42`). Use when path arguments contain digits |
| `--wd` | off | Force Working Dir mode, skipping PR auto-detection |
| `--no-codex` | off | Disable Codex integration entirely (skip Codex detection and execution) |
| `--codex-review` | off | Use Codex general review (`codex:review`) instead of adversarial review |
| `--codex-both` | off | Run both Codex general review and adversarial review in parallel |

Codex flag precedence: `--no-codex` > `--codex-both` > `--codex-review` > default (adversarial only).
Only one Codex mode flag should be used at a time. If multiple are present, the highest-precedence flag wins.

If `$ARGUMENTS` contains explicit publish intent ("comment 달아", "바로 올려", "게시해", "post it"), treat as `-y`.

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

If `-g` (graph) flag is set, additionally analyze import/dependency relationships:
- Trace imports/exports of changed files
- Identify callers and callees using Grep
- Build a mental dependency map of affected modules

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

---

## Step 2.5: Codex Detection

Determine whether the Codex plugin is available and resolve the Codex execution mode. This step is zero-cost: it relies on the skills already listed in the current session's system prompt — no hooks, environment variables, or CLI checks needed.

### Detection

Check if the skill `codex:codex-cli-runtime` is listed among the available skills in the current session. If yes, the Codex plugin is installed and the companion runtime is available.

Note: `codex:adversarial-review` and `codex:review` have `disable-model-invocation: true`, so they do NOT appear in the session skill list. Detection must use `codex:codex-cli-runtime` instead, which has no such restriction.

### Companion Path Resolution

When Codex is detected, resolve the companion script path at runtime:

```bash
COMPANION=$(find ~/.claude/plugins/cache/openai-codex -name "codex-companion.mjs" 2>/dev/null | sort -V | tail -1)
```

If no companion is found, treat Codex as **disabled**.

### Mode Resolution

| Condition | Codex Mode | Companion Subcommand |
|-----------|-----------|---------------------|
| `--no-codex` flag is set | **disabled** | None |
| Codex not available | **disabled** | None |
| `--codex-both` flag is set | **both** | `review --wait` + `adversarial-review --wait` |
| `--codex-review` flag is set | **review** | `review --wait` |
| Default (no Codex flag) | **adversarial** | `adversarial-review --wait` |

In PR mode, append `--base {baseRefName}` to each subcommand to scope the review to the PR diff.

Store the resolved mode for use in Steps 3, 4, and 5. Note: the companion path is resolved here for validation only. Each spawned Codex agent re-resolves the path independently via `find` since agents run in separate contexts. If mode is **disabled**, skip all Codex-related logic in subsequent steps and proceed exactly as before (full backward compatibility).

---

## Step 3: Domain Agents (Parallel Execution)

Launch activated domain agents in parallel. Each agent receives the full diff and context from Step 1.

### Execution Mode

| Mode | Condition | Description |
|------|-----------|-------------|
| **Team agents** (default) | `-s`/`--sub` flag NOT set | Each domain runs as an independent team agent with its own context window. Better result quality for large diffs. |
| **Sub-agents** | `-s`/`--sub` flag set | Each domain runs as a sub-agent via the Agent tool. Results return to the main context. Lighter for small diffs. |

### Team Agent Mode (default)

Used when `-s`/`--sub` flag is NOT set. Each domain agent gets its own context window.

1. `TeamCreate` — name: `code-review`.
2. For each activated domain:
   - `TaskCreate` — subject: `"{Domain} domain analysis"`. Description: changed files relevant to this domain and domain-specific focus areas.
   - Spawn teammate via `Agent` with `team_name: "code-review"`, `name` set to domain name (lowercase, e.g., `"security"`, `"architecture"`), `subagent_type` per Domain Definitions. Prompt includes: full diff from Step 1, changed file context, domain focus areas, and finding format (title, file path, occurrence count, description, suggested fix).
   - `TaskUpdate` — set `owner` to the agent name.
3. Monitor `TaskList` until all domain tasks complete. Collect findings from agent messages.
4. Shut down agents via `SendMessage` with `shutdown_request`.
5. Pass aggregated findings to Step 4.

Fallback: if `TeamCreate` is unavailable, switch to Sub-agent mode.

### Sub-agent Mode

Used when `-s`/`--sub` flag IS set, or as fallback. Launch each domain agent via `Agent` in parallel. Results return to the main context.

### Domain Definitions

Each domain agent — whether team or sub — receives the same prompt structure: diff, changed file contents, and domain-specific analysis instructions.

#### 🛡️ Security

Agent type: `oh-my-claudecode:security-reviewer`

Focus areas:
- Authentication/authorization flaws
- Input validation gaps
- SQL injection, XSS, command injection
- Secret/credential exposure
- Dependency vulnerabilities (if package files changed)
- Insecure configurations

#### ⚡ Performance

Agent type: `oh-my-claudecode:scientist`

Focus areas:
- Algorithm complexity issues (O(n^2) or worse in hot paths)
- N+1 query patterns
- Missing caching opportunities
- Memory leaks or excessive allocation
- Unoptimized database queries
- Blocking operations in async contexts

#### 🏗️ Architecture

Agent type: `oh-my-claudecode:architect`

Focus areas:
- SOLID principle violations
- Coupling/cohesion issues
- Interface contract breakage
- Technical debt introduction
- Inconsistency with existing patterns
- Missing abstractions or over-abstraction

#### 🔍 Domain Logic

Agent type: `oh-my-claudecode:code-reviewer`

Focus areas:
- Business rule correctness
- Error handling completeness
- Edge case coverage
- State management issues
- Race conditions
- Type safety gaps

Each finding must include: title, file path, **primary line number**, occurrence count, description, and suggested fix.

**Primary line number**: The line number in the new version of the file where the issue is most clearly visible. Required for inline comment targeting in PR mode. In the finding's description text, reference locations by section heading, function name, or code pattern — not by raw line number (line numbers are used only for comment placement, not in the output text).

### Codex Parallel Execution

If Codex mode (from Step 2.5) is NOT **disabled**, launch Codex alongside the domain agents in parallel. Codex runs as an independent second reviewer — it is NOT a domain agent, but its execution is concurrent with domain agents.

Codex is invoked via the companion runtime directly through Bash, NOT via `Skill("codex:...")`. The `codex:adversarial-review` and `codex:review` skills have `disable-model-invocation: true`, which blocks `Skill()` invocation. The companion runtime (`codex-companion.mjs`) has no such restriction and is the official programmatic interface for invoking Codex from Claude Code.

#### Team Agent Mode (default, `-s`/`--sub` NOT set)

For each Codex subcommand to invoke (per the mode table in Step 2.5):

1. `TaskCreate` — subject: `"Codex {mode} review"` (e.g., `"Codex adversarial review"`).
2. Spawn teammate via `Agent` with:
   - `team_name: "code-review"`
   - `name: "codex"` (or `"codex-review"` / `"codex-adversarial"` when mode is **both**, to distinguish the two)
   - Prompt: Instruct the agent to run the companion via Bash and return the findings. The Bash command:
     ```
     COMPANION=$(find ~/.claude/plugins/cache/openai-codex -name "codex-companion.mjs" 2>/dev/null | sort -V | tail -1) && node "$COMPANION" {subcommand} --wait
     ```
     Where `{subcommand}` is `adversarial-review` or `review` per the mode table. Use `--base {baseRefName}` for PR mode.
3. `TaskUpdate` — set `owner` to the agent name.

The Codex agent(s) run in parallel with domain agents (Security, Architecture, etc.) within the same `code-review` team. They appear in `TaskList` output alongside domain agents, satisfying traceability (Acceptance Criterion F).

#### Sub-agent Mode (`-s`/`--sub` set)

For each Codex subcommand to invoke:

- Launch via `Agent` with `name: "codex"` (or `"codex-review"` / `"codex-adversarial"` for **both** mode). No `team_name`. The agent runs the companion via Bash (same command as Team Agent Mode) and returns findings.

Launch Codex agents at the same time as domain agents — do NOT wait for domain agents to finish first.

### Codex Failure Handling

If a Codex agent reports a non-zero exit code or returns an error (e.g., quota exhausted, authentication failure, network error):

1. **Do NOT retry** — treat the Codex contribution as unavailable for this run.
2. **Findings = empty** — proceed with domain agent findings only. Do not attempt to parse error output as findings.
3. **Terminal notice** — append `ℹ️ Codex: unavailable (skipped)` to the Terminal format output (see Step 5).
4. **GitHub format** — do NOT include any Codex failure notice. Codex availability is an internal infrastructure detail, not relevant to PR reviewers.

### Fallback (non-Claude Code runners)

If neither team agents nor sub-agents are available (e.g., Codex CLI, Gemini CLI as the runner platform), perform all domain analyses sequentially in a single pass. Analyze each domain's focus areas one by one and collect findings.

---

## Step 4: Cross-Validation

This is the quality gate. Review ALL findings from domain agents against the full context to filter false positives.

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

### Severity Levels

| Severity | Icon | Criteria |
|----------|------|----------|
| Critical | 🔴 | Security vulnerability, data loss risk, crash-inducing bug |
| Warning | 🟡 | Potential bug, performance issue, maintainability concern |
| Info | 🟢 | Suggestion, minor improvement, style note |

### Source Tags

Each finding displays a source tag after the title (e.g., `**Finding title** — Security`). Domain agent findings use their domain name. Codex findings use tags based on the active Codex mode:

| Codex Mode | Source(s) | Tag(s) |
|-----------|-----------|--------|
| **disabled** | Domain agents only | Domain name (e.g., `Security`, `Architecture`) |
| **adversarial** (default) | Domain agents + Codex adversarial | Domain name / `Codex` |
| **review** (`--codex-review`) | Domain agents + Codex review | Domain name / `Codex` |
| **both** (`--codex-both`) | Domain agents + Codex review + Codex adversarial | Domain name / `Codex` (review findings) / `Codex Adv` (adversarial findings) |

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

---

**I2. {finding title}** — {domain}
`{file}` ({N}곳)

{description}
```

#### GitHub Format: Review Summary

Posted as the pull request review `body`. Contains the overview table and any findings that could not be mapped to diff lines (unmapped findings).

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

{if unmapped findings exist:}

---

### 📋 General Findings

{for each unmapped finding, sorted by severity:}

{severity_icon} **{finding title}** — {domain}
`{file}` ({N}곳)

{description}

> **Fix**: {suggestion}

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

{if concrete code change:}
```suggestion
{suggested code — only the replacement lines, matching GitHub suggestion block format}
```
{else:}
> **Fix**: {suggestion}
{end if}
````

Numbering uses the same scheme as Terminal format: `C{n}` / `W{n}` / `I{n}`, starting from 1 per severity. Terminal preview and GitHub inline comments share the same numbers for cross-reference.

For contextual match findings (mapped via contextual fallback):

````markdown
{severity_icon} **{prefix}{n}. {finding title}** — {domain}

📍 이 변경의 영향: `{affected_file}` ({N}곳)

{description}

{if concrete code change:}
```suggestion
{suggested code — only the replacement lines, matching GitHub suggestion block format}
```
{else:}
> **Fix**: {suggestion}
{end if}
````

Use GitHub `suggestion` blocks when the fix is a concrete, localized code change (variable rename, parameter addition, line replacement). The suggestion block enables one-click "Apply suggestion" in GitHub's UI. Use `> **Fix**: ...` for directional or structural suggestions that span multiple locations.

### Formatting Rules

**Common rules (both formats)**:
- Location: file path on its own line, occurrence count only (e.g., "(3곳)", "(1곳)"). No line numbers (`L42`, `L58` 등) anywhere — descriptions and Fix suggestions reference locations by section heading, function name, or searchable code pattern instead.
- Finding title must NOT repeat the file name (location is already on its own line).
- Omit severity sections that have 0 findings.
- **Bullet management**: If a single finding has more than 5 sub-points, consolidate.
- **Commit SHA references**: Never use backticks around SHAs. Use plain text or markdown links.

**Terminal-specific rules**:
- No `<details>` or HTML tags — they don't render in CLI.
- Summary line with severity icons: `Findings: 🔴 1 critical · 🟡 3 warnings · 🟢 1 info`.
- Severity headers use `### < {icon} {Severity} ({n}) >` format with `< >` brackets.
- Severity sections are separated by `────────────────────` (Unicode thin box drawing × 20).
- Severity header is followed by `---` before the first finding. Findings within the same severity are also separated by `---`.
- Each finding is numbered with a severity prefix: `C{n}` (Critical), `W{n}` (Warning), `I{n}` (Info), starting from 1 per severity.
- Finding title first (bold — renders bright in CLI), file path second.
- Fix is always natural language in a blockquote (`> **Fix**: ...`), referencing by section/pattern.
- Domains with no findings: omit entirely (no "✅ ... No issues found" line).
- Codex failure notice (if applicable): append `ℹ️ Codex: unavailable (skipped)` after the summary line. Only shown when Codex mode was NOT disabled but companion returned non-zero exit. Do NOT include in GitHub format.

**GitHub-specific rules (inline review comments)**:
- Review Summary: severity counts and key changes at the top. Unmapped findings (after contextual mapping) included below under "General Findings" as fallback only.
- Inline Comment: one finding per comment. No `<details>` tags — each comment is self-contained.
- Fix format in inline comments:
  - Concrete, localized code change → GitHub `suggestion` block (enables one-click apply).
  - Directional or multi-location suggestion → `> **Fix**: ...` natural language.
- Domains with no findings: omit entirely from both summary and inline comments.

---

## Step 6: Publisher

Based on the mode and flags, publish the review output.

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
   - Dead code from a removal → map to the diff line where the related code was removed
   Include as an inline comment at the contextual location. Prepend to the comment body:
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
- `body`: Review Summary template (severity table + unmapped findings if any + footer)
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

1. Parse `$ARGUMENTS` to determine mode (PR / Working Dir / Path) and flags (including Codex flags).
2. **Context Builder**: Gather diff, commit history, related files, and PR description (if applicable).
3. **Domain Router**: Analyze changed file types and activate relevant domains. Respect `--domain` override.
4. **Codex Detection**: Check `codex:codex-cli-runtime` availability, resolve companion path, and determine Codex mode (adversarial / review / both / disabled).
5. **Domain Agents + Codex**: Launch activated domain agents in parallel. If Codex is enabled, launch Codex agent(s) via companion runtime concurrently. Collect all findings. If Codex fails (non-zero exit), proceed with domain findings only.
6. **Cross-Validation**: Verify each finding (domain + Codex) against expanded context, git history, comments, and PR intent. Classify as Confirmed / Demoted / Dismissed.
7. **Output Generator**: Produce severity-first structured output with source tags (domain names + Codex tags per mode).
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
- When Agent tool is unavailable or subagent types are not registered, perform all analyses sequentially as a single-pass fallback.
