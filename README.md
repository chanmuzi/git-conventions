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

### Option 1: Skills CLI (All Agents)

```bash
npx skills add chanmuzi/git-conventions
```

The interactive installer will guide you through:

1. **Select skills to install** — use `space` to toggle each skill
   - `commit` — Create a git commit following project conventions
   - `pr` — Create a pull request following project conventions
   - `issue` — Create a GitHub issue with templates and auto-labeling
   - `review-reply` — Review AI-generated PR review comments and reply
   - `code-review` — Context-aware multi-agent code review with severity-based findings
2. **Which agents do you want to install to?** — select your agents (e.g. Codex, Cursor, Gemini CLI, GitHub Copilot, …)
3. **Installation scope** — `Project` (current repo only) or `Global` (available across all projects)
4. **Installation method** — `Symlink` (recommended) or `Copy`
5. **Confirm** and done

> To install all skills at once without prompts:
> ```bash
> npx skills add chanmuzi/git-conventions --skill commit --skill pr --skill issue --skill review-reply --skill code-review -g
> ```

### Option 2: Claude Code Plugin

```bash
# Add marketplace
/plugin marketplace add chanmuzi/git-conventions

# Install plugin
/plugin install git-conventions@git-conventions
```

## Update

> **Recommended:** Enable auto-update to receive new features and fixes automatically.
> See platform-specific instructions below.

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

### Claude Code Plugin

```bash
# Update the plugin
/plugin update git-conventions@git-conventions
```

Or enable auto-update for hands-free upgrades (recommended):

`/plugin` → **Marketplaces** tab → select marketplace → **Enable auto-update**

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
**Auto-labeling:** `type: bug`, `type: feature`, `type: enhancement`, `priority: critical/high/medium/low`, etc.

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
- `--inline` — Add inline comments on PR (PR mode only)
- `-y` / `-f` — Publish without approval
- `-g` — Enable code graph analysis

**Severity levels:** 🔴 Critical, 🟡 Warning, 🟢 Info

## Language Behavior

All commands write output (commit messages, PR titles/body) in the language configured in your project's `CLAUDE.md`. If no language is set, the user's conversational language is used.

Technical terms are kept in their original form to preserve clarity — no forced translations.

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
