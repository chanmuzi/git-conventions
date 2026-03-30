---
name: code-review
description: |
  Review code changes using context-aware multi-agent pipeline with severity-based findings.
  TRIGGER when: user asks to review code, analyze PR quality, check for issues, run code review, or audit changes (e.g., "코드 리뷰해줘", "review this PR", "코드 분석해줘", "리뷰 돌려줘").
  DO NOT TRIGGER when: user is replying to review comments (use review-reply), creating PRs, committing, or performing git operations without review intent.
argument-hint: "[PR번호|경로] [--domain security,perf] [--inline] [-y] [-g]"
version: "1.3.0"
allowed-tools: Bash(git *), Bash(gh *), Read, Grep, Glob, Agent
---

## Parse Arguments

Parse `$ARGUMENTS` to determine the review mode and flags.

### Mode Detection

| Mode | Condition | Description |
|------|-----------|-------------|
| **PR Review** | `$ARGUMENTS` contains a number or `--pr` | Review a specific PR's diff + commit context |
| **Working Dir** | `$ARGUMENTS` is empty (no arguments) | Review current working directory changes (staged + unstaged) |
| **Path Review** | `$ARGUMENTS` contains a file/directory path | Review code at the specified path |

### Flag Detection

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--yes` | `-y` | off | Publish without approval |
| `--force` | `-f` | off | Same as `-y` |
| `--graph` | `-g` | off | Enable code graph analysis (dependency map) |
| `--inline` | — | off | Add inline comments on PR (PR mode only) |
| `--domain` | `-d` | auto | Override domain selection (e.g., `--domain security,perf`) |

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

## Step 3: Domain Agents (Parallel Execution)

Launch activated domain agents in parallel using the Agent tool. Each agent receives the full diff and context from Step 1.

### Agent Definitions

#### Security Agent

```
subagent_type: "oh-my-claudecode:security-reviewer"
```

Prompt the agent with: the diff, changed file contents, and instruction to analyze for:
- Authentication/authorization flaws
- Input validation gaps
- SQL injection, XSS, command injection
- Secret/credential exposure
- Dependency vulnerabilities (if package files changed)
- Insecure configurations

Each finding must include: title, file:line, description, and suggested fix.

#### Performance Agent

```
subagent_type: "oh-my-claudecode:scientist"
```

Prompt the agent with: the diff, changed file contents, and instruction to analyze for:
- Algorithm complexity issues (O(n^2) or worse in hot paths)
- N+1 query patterns
- Missing caching opportunities
- Memory leaks or excessive allocation
- Unoptimized database queries
- Blocking operations in async contexts

Each finding must include: title, file:line, description, and suggested fix.

#### Architecture Agent

```
subagent_type: "oh-my-claudecode:architect"
```

Prompt the agent with: the diff, changed file contents, project structure, and instruction to analyze for:
- SOLID principle violations
- Coupling/cohesion issues
- Interface contract breakage
- Technical debt introduction
- Inconsistency with existing patterns
- Missing abstractions or over-abstraction

Each finding must include: title, file:line, description, and suggested fix.

#### Domain Logic Agent

```
subagent_type: "oh-my-claudecode:code-reviewer"
```

Prompt the agent with: the diff, changed file contents, and instruction to analyze for:
- Business rule correctness
- Error handling completeness
- Edge case coverage
- State management issues
- Race conditions
- Type safety gaps

Each finding must include: title, file:line, description, and suggested fix.

### Fallback (non-Agent runners)

If Agent tool is unavailable (non-Claude Code runners) or the specified subagent types are not registered, perform all domain analyses sequentially in a single pass. Analyze each domain's focus areas one by one and collect findings.

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

---

## Step 5: Output Generator

Produce severity-first structured output in Korean (technical terms in English).

### Severity Levels

| Severity | Icon | Criteria |
|----------|------|----------|
| Critical | 🔴 | Security vulnerability, data loss risk, crash-inducing bug |
| Warning | 🟡 | Potential bug, performance issue, maintainability concern |
| Info | 🟢 | Suggestion, minor improvement, style note |

### Output Template

Write the review in the language configured in the project's CLAUDE.md.
If no language is configured, follow the user's conversational language.
Examples below are in Korean.

```markdown
## Code Review: {target}

**Domains**: {activated domains joined by " • "} | **Findings**: {total count}

| Severity | Count | Domains |
|----------|-------|---------|
| 🔴 Critical | {n} | {domains} |
| 🟡 Warning  | {n} | {domains} |
| 🟢 Info     | {n} | {domains} |

---

### 🔴 Critical

**{finding title}** — {domain}
`{file}:{line}`
{description in Korean}
Suggested fix: {suggestion}

### 🟡 Warning

**{finding title}** — {domain}
`{file}:{line}`
{description in Korean}
Suggested fix: {suggestion}

### 🟢 Info

**{finding title}** — {domain}
`{file}:{line}`
{description in Korean}

---
✅ {domain}: No issues found
```

### Formatting Rules

- If findings exceed 5 per severity group, wrap each group in `<details><summary>` for collapsibility.
- Domains with no findings: show as `✅ {Domain}: No issues found` (one line each, at the bottom).
- Each finding: **Title** — Domain + location + description + suggested fix (for Critical/Warning).
- Summary table always at the top.
- Omit severity sections that have 0 findings.
- **Bullet management**: If a single finding has more than 5 sub-points, consolidate.
- **Commit SHA references**: Never use backticks around SHAs. Use plain text or markdown links.

---

## Step 6: Publisher

Based on the mode and flags, publish the review output.

### Approval Logic

| Condition | Behavior |
|-----------|----------|
| `-y` or `-f` flag | Publish immediately |
| `$ARGUMENTS` contains publish intent | Publish immediately |
| Default | Show output to user, ask for approval before publishing |

### PR Review Mode — Publishing

**Summary comment** (always): Post the full review as a PR comment.

```
gh pr comment {number} --body "$(cat <<'EOF'
{review output}

Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

**Inline comments** (only if `--inline` flag): For each Critical and Warning finding with a specific file:line, add an inline review comment.

Use the GitHub pull request review API to submit all inline comments as a single review:

```
gh api repos/{owner}/{repo}/pulls/{number}/reviews --input - <<'EOF'
{"event":"COMMENT","body":"Code review by Claude Code","comments":[{"path":"{file}","line":{line},"body":"{finding_summary}"}]}
EOF
```

If inline comments fail (e.g., line not in diff), fall back to the summary comment only. Do not retry individual inline comments.

### Working Dir / Path Mode — Publishing

Display the review output directly to the user in the conversation. No GitHub publishing.

---

## Task

1. Parse `$ARGUMENTS` to determine mode (PR / Working Dir / Path) and flags.
2. **Context Builder**: Gather diff, commit history, related files, and PR description (if applicable).
3. **Domain Router**: Analyze changed file types and activate relevant domains. Respect `--domain` override.
4. **Domain Agents**: Launch activated agents in parallel via Agent tool. Collect all findings.
5. **Cross-Validation**: Verify each finding against expanded context, git history, comments, and PR intent. Classify as Confirmed / Demoted / Dismissed.
6. **Output Generator**: Produce severity-first structured output. Apply formatting rules.
7. **Publisher**: Based on mode and flags, publish or display the review.

**Important:**
- Do NOT publish review comments without user approval unless `-y`/`-f` is set or explicit publish intent is detected.
- Do NOT replace linter checks — focus on semantic, architectural, and logic issues.
- Do NOT suggest auto-fix or auto-merge — this is review only.
- This skill is independent from `review-reply`. `code-review` generates reviews (proactive); `review-reply` responds to received reviews (reactive).
- **Assignee**: If creating GitHub PRs or issues, always include `--assignee @me`.
- **Commit references**: Never wrap commit SHAs in backticks. Use plain text or explicit markdown links.
- Adapt output language to the project's CLAUDE.md language setting.
- When Agent tool is unavailable or subagent types are not registered, perform all analyses sequentially as a single-pass fallback.
