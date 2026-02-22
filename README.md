# git-conventions

English | [한국어](README.ko.md)

A Claude Code skill that enforces consistent Git commit messages, PR templates, and AI review analysis across projects.

## Installation

```bash
# Add marketplace
/plugin marketplace add chanmuzi/git-conventions

# Install plugin
/plugin install git-conventions@git-conventions
```

## Skills

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

### `/review` — Review AI PR Comments

Collects AI-generated review comments (CodeRabbit, Copilot, etc.) from a PR, analyzes their validity against the actual code, and discusses findings with you.

```
/review          # Review current branch's PR
/review 42       # Review PR #42
```

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
- Use `/commit`, `/pr`, `/pr release`, `/review` commands for full workflows
```

## License

MIT
