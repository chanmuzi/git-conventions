<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:6366F1,50:8B5CF6,100:A855F7&height=220&section=header&text=git-conventions&fontSize=52&fontColor=ffffff&fontAlignY=38&desc=Agent%20Skills%20for%20consistent%20Git%20workflows&descSize=18&descColor=E2E8F0&descAlignY=58&animation=fadeIn" alt="git-conventions" width="100%" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/skills-5-8B5CF6?logo=git&logoColor=white" alt="Skills" />
  <a href="https://agentskills.io"><img src="https://img.shields.io/badge/Agent_Skills-compatible-0EA5E9?logo=robotframework&logoColor=white" alt="Agent Skills" /></a>
  <img src="https://img.shields.io/github/license/chanmuzi/git-conventions?color=blue" alt="License" />
  <img src="https://img.shields.io/github/last-commit/chanmuzi/git-conventions?color=orange" alt="Last Commit" />
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
/plugin marketplace add chanmuzi/git-conventions

# Install plugin
/plugin install git-conventions@git-conventions
```

### Option 2: Skills CLI (All Agents)

```bash
npx skills add chanmuzi/git-conventions
```

**Included skills:**

| Skill | Description |
|-------|-------------|
| `commit` | Create a git commit following project conventions |
| `pr` | Create a pull request following project conventions |
| `issue` | Create a GitHub issue with templates and auto-labeling |
| `review-reply` | Review AI-generated PR review comments and reply |
| `code-review` | Context-aware multi-agent code review with severity-based findings |

The interactive installer lets you select skills, target agents, scope (project/global), and install method.

> To install all skills at once without prompts:
> ```bash
> npx skills add chanmuzi/git-conventions --skill commit --skill pr --skill issue --skill review-reply --skill code-review -g
> ```

## Update

> **Recommended:** Enable auto-update to receive new features and fixes automatically.
> See platform-specific instructions below.

### Claude Code Plugin

```bash
# Update the plugin
/plugin update git-conventions@git-conventions
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

Analyzes your staged/unstaged changes and proposes a commit message following the conventional commit format.

```
/commit
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
- `--no-codex` — Disable Codex integration
- `--codex-review` — Use Codex general review instead of adversarial
- `--codex-both` — Run both Codex review and adversarial review

**Natural language:** You can use flags or natural language — Claude translates your intent into the appropriate flags before invoking the skill.

```
/code-review 42 바로 올려줘             # → /code-review 42 -y
/code-review security만 봐줘            # → /code-review --domain security
/code-review codex 둘 다 돌려줘         # → /code-review --codex-both
/code-review codex 없이 리뷰해줘        # → /code-review --no-codex
/code-review working dir로 봐줘         # → /code-review --wd
```

**Codex integration:** When the [Codex plugin](https://github.com/anthropics/codex) is installed, `/code-review` automatically runs Codex adversarial review in parallel with domain agents. Findings are cross-validated and merged with source tags. Use `--no-codex` to opt out.

**Inline review comments:** In PR mode, findings are published as inline review comments attached to specific diff lines via the GitHub Review API. Each finding becomes an independent conversation thread — reply, resolve, or apply suggestions individually. A severity summary table is included in the review body. Findings outside the diff are included as "General Findings" in the summary.

**Severity levels:** 🔴 Critical, 🟡 Warning, 🟢 Info

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
- Use `/commit`, `/pr`, `/pr release`, `/issue`, `/review-reply`, `/code-review` commands for full workflows
```

## License

MIT
