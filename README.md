<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:6366F1,50:8B5CF6,100:A855F7&height=220&section=header&text=git-claw&fontSize=52&fontColor=ffffff&fontAlignY=38&desc=Agent%20Skills%20for%20consistent%20Git%20workflows&descSize=18&descColor=E2E8F0&descAlignY=58&animation=fadeIn" alt="git-claw" width="100%" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/skills-6-8B5CF6?logo=git&logoColor=white" alt="Skills" />
  <a href="https://agentskills.io"><img src="https://img.shields.io/badge/Agent_Skills-compatible-0EA5E9?logo=robotframework&logoColor=white" alt="Agent Skills" /></a>
  <img src="https://img.shields.io/github/license/chanmuzi/git-claw?color=blue" alt="License" />
  <img src="https://img.shields.io/github/last-commit/chanmuzi/git-claw?color=orange" alt="Last Commit" />
</p>

<p align="center">
  <a href="README.md">English</a> · <a href="README.ko.md">한국어</a>
</p>

---

An [Agent Skill](https://agentskills.io) that enforces consistent Git commit messages, PR templates, and AI review analysis across projects.

Works with Claude Code, Codex CLI, Gemini CLI, Cursor, GitHub Copilot, Amp, and [40+ agents](https://skills.sh).

## Installation

### Option 1: Claude Code Plugin (Recommended)

```bash
# Add marketplace
/plugin marketplace add chanmuzi/git-claw

# Install plugin
/plugin install git-claw@git-claw
```

### Option 2: Skills CLI (All Agents)

```bash
npx skills add chanmuzi/git-claw
```

**Included skills:**

| Skill | Description |
|-------|-------------|
| `commit` | Create a git commit following project conventions |
| `pr` | Create a pull request following project conventions |
| `issue` | Create a GitHub issue with templates and auto-labeling |
| `review-reply` | Review AI-generated PR review comments and reply |
| `code-review` | Context-aware multi-agent code review with severity-based findings |
| `handoff` | Generate a copy-ready handoff prompt for session transfer |

The interactive installer lets you select skills, target agents, scope (project/global), and install method.

> To install all skills at once without prompts:
> ```bash
> npx skills add chanmuzi/git-claw --skill commit --skill pr --skill issue --skill review-reply --skill code-review --skill handoff -g
> ```

## Update

> **Recommended:** Enable auto-update to receive new features and fixes automatically.
> See platform-specific instructions below.

### Claude Code Plugin

```bash
# Update the plugin
/plugin update git-claw@git-claw
```

Or enable auto-update for hands-free upgrades (recommended):

`/plugin` → **Marketplaces** tab → select marketplace → **Enable auto-update**

### Skills CLI

```bash
# Check for available updates
npx skills check

# Update all installed skills
npx skills update
```

> If you installed with Symlink (recommended), a single update applies to all agents at once. Copy installs require updating each copy individually.
>
> Skills CLI does not auto-update. Run `npx skills check` periodically, or after major releases announced in the [changelog](CHANGELOG.md).

## Skills

> **Skill prefix by agent:** Claude Code uses `/` (e.g., `/commit`), Codex CLI uses `$` (e.g., `$commit`).

### `/commit` — Create a Git Commit

Analyzes your staged/unstaged changes and proposes a commit message following the conventional commit format. Automatically detects when the current changes overlap with the last commit and amends instead of creating a new commit.

```
/commit              # Analyze and commit (with amend auto-detection)
/commit --no-amend   # Skip amend detection; always create a new commit
```

**Commit message format:**
```
{type}: {description}
{type}({scope}): {description}
```

Types: `feat`, `fix`, `refactor`, `style`, `docs`, `test`, `perf`, `chore`, `hotfix`

### `/pr` — Create a Pull Request

Creates a PR with a structured template. Automatically detects whether to use the Individual PR template or the Release PR template.

```
/pr              # Individual PR (Feat, Fix, Refactor, etc.)
/pr -g           # Include Mermaid change-flow graph
/pr release      # Release PR (dev → main integration)
```

**PR title format:**
```
{Type}: {description}               # Individual (e.g., Feat: add new feature)
Release: dev → main 통합 (vX.Y.Z)  # Release
```

### `/issue` — Create a GitHub Issue

Creates a structured issue with the appropriate template and auto-assigns type/priority labels.

```
/issue             # General issue (type inferred from context)
/issue bug         # Bug report
/issue feature     # Feature request
```

**Templates:** Bug Report, Feature Request, General
**Auto-labeling:** `bug`, `feature`, `enhancement`, `critical`/`high`/`medium`/`low`, etc.

### `/review-reply` — Review & Reply to PR Comments

Collects review comments (CodeRabbit, Copilot, teammates, etc.) from a PR, analyzes their validity against the actual code, and discusses findings with you.

```
/review-reply          # Review current branch's PR
/review-reply 42       # Review PR #42
```

### `/code-review` — Context-Aware Code Review

Analyzes PR or local code changes using a multi-agent pipeline with domain-specific agents (Security, Performance, Architecture, Domain Logic). Cross-validates findings against code context to filter false positives, then produces severity-based structured output.

```
/code-review           # Auto-detect PR on current branch, or review working dir
/code-review 42        # Review PR #42
/code-review src/auth/ # Review specific path
/code-review --wd      # Force working dir review (skip PR auto-detection)
```

**Flags:**
- `--wd` — Force working directory mode, even on a PR branch
- `--domain security,perf` — Override auto-detected domains
- `-y` / `-f` — Publish without approval
- `-g` — Generate Mermaid change-flow graph (PR mode only)
- `-q` / `--quick` — Quick mode: single-pass analysis (no agent spawn), max 2 domains, Critical/Warning first (Info fallback when none found)
- `--no-codex` — Disable Codex integration
- `--codex` — Run both Codex review and adversarial review
- `--codex-general` — Use Codex general review only (without adversarial)

**Natural language:** You can use flags or natural language — Claude translates your intent into the appropriate flags before invoking the skill.

```
/code-review 42 바로 올려줘             # → /code-review 42 -y
/code-review security만 봐줘            # → /code-review --domain security
/code-review codex 둘 다 돌려줘         # → /code-review --codex
/code-review codex 없이 리뷰해줘        # → /code-review --no-codex
/code-review working dir로 봐줘         # → /code-review --wd
/code-review 간단하게 봐줘              # → /code-review --quick
```

**Codex integration (optional, recommended):** When the [Codex plugin](https://github.com/openai/codex) is installed and authenticated, `/code-review` automatically runs Codex adversarial review in parallel with domain agents. Findings are cross-validated and merged with source tags. Use `--no-codex` to opt out.

To set up Codex integration:
1. Install the Codex plugin: `claude plugin add codex`
2. Run `!codex setup` in Claude Code to authenticate with your OpenAI API key
3. `/code-review` will automatically detect and use Codex — no additional flags needed

**Inline review comments:** In PR mode, findings are published as inline review comments attached to specific diff lines via the GitHub Review API. Each finding becomes an independent conversation thread — reply, resolve, or apply suggestions individually. A severity summary table is included in the review body. Findings outside the diff are included as "General Findings" in the summary.

**Severity levels:** 🔴 Critical, 🟡 Warning, 🟢 Info

### `/handoff` — Session Handoff Prompt

Generates a copy-ready handoff prompt that transfers work context to a new session. Auto-detects artifacts, git state, and conversation context, then structures a concise prompt following the **reference-not-repeat** principle.

```
/handoff                   # Auto-detect and generate handoff prompt
/handoff -y                # Skip confirmation, output directly
/handoff auth refactoring  # Focus handoff on a specific topic
```

**Detection cascade:** Artifacts (`.omc/specs/`, `.omc/plans/`) → Git state → Conversation context

**Skill recommendation:** When plugins like OMC or Codex are detected, the handoff recommends the most appropriate skill for the next session (e.g., `/autopilot`, `/ralph`, `/commit`).

**Output:** Terminal text optimized for `/copy`. Never repeats artifact content — references file paths instead.

## Language Behavior

All commands write output (commit messages, PR titles/body) in the language configured in your project's `CLAUDE.md`. If no language is set, the user's conversational language is used.

Technical terms are kept in their original form to preserve clarity — no forced translations.

## Label System

`/pr` and `/issue` share a unified label scheme. Labels are auto-created on first use — if a label already exists, it is left unchanged.

| Label | Color | Source |
|-------|-------|--------|
| `bug` | 🔴 `d73a4a` | GitHub default |
| `feature` | 🔵 `0075ca` | GitHub default |
| `enhancement` | 🩵 `a2eeef` | GitHub default |
| `docs` | 🟣 `5319e7` | GitHub default |
| `chore` | 🟡 `e4e669` | Standard |
| `refactor` | 🟪 `d4c5f9` | Standard |
| `test` | 🟢 `bfd4f2` | Standard |
| `perf` | 🟠 `f9d0c4` | Standard |
| `hotfix` | 🔴 `b60205` | Standard |
| `release` | 🔵 `1d76db` | Standard |
| `critical` | 🔴 `b60205` | Priority |
| `high` | 🟠 `d93f0b` | Priority |
| `medium` | 🟡 `fbca04` | Priority |
| `low` | 🟢 `0e8a16` | Priority |

- **No namespace prefix** — clean, simple names (`bug` instead of `type: bug`)
- **GitHub-standard colors** — matches GitHub's default palette where applicable
- **Conflict-safe** — `gh label create ... 2>/dev/null || true` (create if missing, skip if exists)

## Conventions at a Glance

| Item | Format | Example |
|------|--------|---------|
| Commit | `{type}: {desc}` (lowercase) | `feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Branch | `{type}/{kebab-case}` (English) | `feat/multiturn-context-persistence` |
| PR Title | `{Type}: {desc}` (capitalized) | `Feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Release PR | `Release: dev → main 통합 (vX.Y.Z)` | `Release: dev → main 통합 (v0.4.1)` |

## Integration with CLAUDE.md

Add the following to your global `~/.claude/CLAUDE.md` to reference these conventions:

```markdown
## Git Conventions
- Commit: `{type}: {description}` (lowercase prefix: feat, fix, refactor, style, docs, test, perf, chore, hotfix)
- Branch: `{type}/{english-kebab-case}` (feat/, fix/, refactor/, docs/, hotfix/)
- PR title: `{Type}: {description}` (capitalized prefix: Feat, Fix, Refactor, Perf, etc.)
- Release PR: `Release: dev → main 통합 (vX.Y.Z)`
- Use `/commit`, `/pr`, `/pr release`, `/issue`, `/review-reply`, `/code-review`, `/handoff` commands for full workflows
```

## License

MIT
