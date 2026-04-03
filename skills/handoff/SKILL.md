---
name: handoff
description: >-
  Generate a copy-ready handoff prompt for transferring work context to a new session.
  TRIGGER when: user asks to hand off work, create a handoff prompt, transfer context, wrap up session, prepare for next session (e.g., "handoff 해줘", "다음 세션으로 넘겨줘", "작업 이관해줘", "handoff prompt 만들어줘").
  DO NOT TRIGGER when: user is committing, creating PRs, reviewing code, or performing other git operations without handoff intent.
version: "1.8.1"
allowed-tools: Bash(git *), Read, Glob, Grep
---

## Core Principle

> **Never repeat artifact content.** If a spec, plan, or document exists, reference its path. The handoff prompt contains only: (1) pointers to existing artifacts, (2) session context not captured elsewhere, (3) a directive for the next session's first action.

## Parse Arguments

Parse `$ARGUMENTS` to determine the handoff mode.

| Argument | Type | Description |
|----------|------|-------------|
| `topic` | Optional string | Focus filter — handoff concentrates on this topic only (e.g., `/handoff auth 리팩토링`) |
| `-y` | Optional flag | Skip confirmation — output handoff prompt directly without draft review |

If `$ARGUMENTS` contains `-y`, extract it as the skip-confirmation flag. The remaining text (if any) is the topic filter.

If `$ARGUMENTS` is empty, proceed with auto-detection and default confirmation flow.

## Detection Cascade

Run the following detection layers in priority order. Each layer enriches the handoff context.

---

**Layer 0 — Topic Override**

If a topic was provided in `$ARGUMENTS`, use it to scope the handoff. The topic guides which parts of the session context to include. Continue to subsequent layers for context detection, but filter all results through this topic.

---

**Layer 1 — Sentinel Skill Detection (Zero-cost)**

Check if these sentinel skill names exist in the current session's available skills. This is zero-cost: no Bash, no filesystem reads — check the session's context window for skill names.

| Sentinel | Plugin | What it enables |
|----------|--------|-----------------|
| `oh-my-claudecode:autopilot` | OMC | Recommend `/autopilot` or `/ralph` for spec/plan-based execution |
| `git-claw:commit` | git-claw (self) | Recommend `/commit` for uncommitted changes |
| `git-claw:pr` | git-claw (self) | Recommend `/pr` for committed changes without PR |

Store detected sentinels for use in the skill recommendation step.

---

**Layer 2 — Conversation Context Analysis**

Analyze the current conversation to understand what this session is about. This is the **primary context source** — it always runs and is never a fallback.

Extract:
- What task was the user working on?
- What key decisions were made?
- What was completed, and what remains?
- What direction was agreed upon?

The output of this layer scopes all subsequent layers — Layer 3 (Git) and Layer 4 (Artifact Search) are filtered through this understanding.

If a topic filter was provided (Layer 0), focus the analysis accordingly.

**This layer triggers the confirmation flow** (unless `-y` flag is set or a relevant artifact is found in Layer 4), because conversation-derived summaries need user validation before handoff.

---

**Layer 3 — Git State Detection**

Gather current git state:

```bash
git branch --show-current
git status --short
git diff --stat
git log --oneline -5
git stash list
```

Use this to populate the handoff `상황` section:
- Current branch name
- Uncommitted changes (count and nature)
- Recent commits (what was done this session)
- Stashed work (if any)

---

**Layer 4 — Artifact Search (Context-Scoped)**

Search for existing structured artifacts using the Glob tool:

```
Glob: .omc/specs/*.md
Glob: .omc/plans/*.md
```

Combine results from both patterns.

**Important:** Only select artifacts that are **relevant to the conversation context identified in Layer 2**. Artifacts from other sessions or unrelated work MUST be ignored. The conversation context is the filter — not the other way around.

If relevant artifacts are found:
- Read the first few lines **only to verify relevance** to the current session's work.
- Do **not** copy or quote any artifact content into the handoff; reference the artifact **by its path only**.
- If **multiple** artifacts appear relevant, use the conversation context (Layer 2) to autonomously select the best match — the conversation typically mentions which artifact was created or worked on. Only ask the user if the model genuinely cannot determine the most relevant one after analysis.
- This is the **artifact-present** path. Skip confirmation — generate the final handoff prompt directly.

If no relevant artifacts are found, proceed with conversation-derived context from Layer 2.

## Build Handoff Prompt

Construct the handoff using the two-zone output structure.

### Zone 1 — Meta Zone (terminal display only)

Display detection results to the user for transparency. This zone is NOT part of the handoff prompt and will not be included when the user runs `/copy`.

Show:
- Which artifacts were found (paths)
- Which sentinel skills were detected (plugin names)
- Which skill is recommended and why

### Zone 2 — Handoff Zone (the copy target)

Separate the Meta Zone and Handoff Zone with a clear `---` divider.

**Wrap the entire Handoff Zone in a fenced code block with the language set to `markdown`.** This enables `/copy` to present it as a selectable item in the picker UI, so the user can copy only the handoff content without the Meta Zone.

### Directive Line

The handoff prompt always opens with a **directive** — 1~3 lines that tell the next session what to do first.

- When the intent is clear: a concrete command (what, where, how)
- When exploration is needed: scope the investigation and specify expected output
- When continuing with a skill: weave the skill name into the directive so copy-paste triggers it directly

The tone does not need to be strictly imperative. Natural phrasing is fine as long as the intended action is unambiguous.

### Section Pool

Below the directive, add structured context by selecting from the section pool. Each section uses a **bold heading** followed by bullets.

| Section | Purpose | When to include |
|---------|---------|-----------------|
| `상황` | Current state, background, git metadata | **Always** |
| `원인` | Diagnosed root cause of a problem | Bug/issue where root cause was identified |
| `진행 상황` | Completed vs remaining work | Work is partially done |
| `판단 필요 사항` | Options and criteria for a pending decision | Direction not yet determined |
| `조사 결과` | Investigation/analysis findings | Research done, execution not started |
| `참고` | Files, PRs, issues, artifact paths, branches | References exist (**almost always**) |

**Composition rules:**
- `상황` is always present. `참고` is present whenever references exist.
- Select 0–2 variable sections (`원인`, `진행 상황`, `판단 필요 사항`, `조사 결과`) based on the scenario. Do not include sections that have no content.
- Keep each section to 5 bullets or fewer. Consolidate or point to artifact paths for detail.
- Omit empty sections entirely.
- Artifact paths from Layer 4 belong in `참고`. Never copy artifact content inline — reference by path only.
- Git state (branch, status) is included in `상황` as inline info, not as a standalone section.

### Scenario Patterns

Typical combinations — adapt based on actual session content, not rigid templates.

**Clear task** — directive + `상황` + `원인` + `참고`:

```markdown
{project}에서 {problem}을 {approach}로 해결할 것.

**상황**
- {background and current state}
- Branch: `{branch}` / Status: {state}

**원인**
- {root cause analysis}

**참고**
- {files, PRs, etc.}
```

**Direction needed** — directive + `상황` + `판단 필요 사항` + `참고`:

```markdown
{project}에서 아래 이슈의 영향 범위를 진단하고
대응 방식을 결정하여 작업 계획을 제시할 것.

**상황**
- {background and current state}

**판단 필요 사항**
- {option A vs option B, criteria}

**참고**
- {files, PRs, etc.}
```

**Partial progress** — directive + `상황` + `진행 상황` + `참고`:

```markdown
{artifact} 참고하여 {next step}부터 이어서 진행할 것.

**상황**
- {background and what this session was about}
- Branch: `{branch}` / Status: {state}

**진행 상황**
- {completed items}
- {remaining items}

**참고**
- Spec: {artifact_path}
```

**Investigation complete** — directive + `상황` + `조사 결과` + `참고`:

```markdown
아래 조사 결과를 바탕으로 {topic}에 대한 구현 방향을
결정하고 작업을 시작할 것.

**상황**
- {background}

**조사 결과**
- {key findings}
- {implications}

**참고**
- {files, PRs, etc.}
```

**Skill continuation (minimal)** — directive + `상황`:

```markdown
/autopilot으로 {artifact} 참고하여 {next step}부터 진행.

**상황**
- {brief context}
- Branch: `{branch}` / Status: {state}
```

### Skill Recommendation Mapping

Based on sentinel detection (Layer 1) + context type (Layers 2-4):

| Context | OMC Available | No OMC | Strategy |
|---------|---------------|--------|----------|
| Spec exists (`.omc/specs/`) | `/autopilot` or `/ralph` | "spec 참고하여 작업 진행" | Spec-driven execution |
| Plan exists (`.omc/plans/`) | `/autopilot` | "plan 참고하여 작업 진행" | Plan-driven execution |
| Code changes (uncommitted) | `/commit` | `/commit` | Stage and commit first |
| Committed, no PR | `/pr` | `/pr` | Create PR |
| Conversation only | Context-dependent | Plain task description | Varies by topic |

**Important:** Weave the skill name into the directive line so that copy-paste triggers the skill directly:
- Good: `/autopilot으로 spec 참고하여 Step 4부터 진행`
- Bad: `Step 4부터 진행 (추천: /autopilot)`

## Confirmation Flow

```
Conversation analyzed → Relevant artifact found? ──yes──→ Output directly
                              │
                              no
                              │
                              ├── -y flag? ──yes──→ Output directly
                              │
                              no
                              │
                              └── Show draft → user reviews and uses `/copy` or requests modifications
```

- **Relevant artifact found**: Skip confirmation. The handoff is a straightforward directive with minimal context sections.
- **No relevant artifact, no -y**: Show draft directly. The user reviews the output and either uses `/copy` to accept, or requests modifications.
- **-y flag**: Always skip confirmation regardless of artifact presence.

Do NOT append confirmation questions like "이 내용으로 handoff 하면 될까요?" — the code block output is self-evident. If the user wants changes, they will ask.

## Output

Output the final handoff prompt to the terminal. The user can then use `/copy` to copy it for the next session.

**Important:**
- Do NOT create any files. The terminal output IS the deliverable.
- Do NOT execute the recommended next action. Only suggest it.
- Do NOT modify any git state (no commit, push, branch creation). This skill is strictly read-only.
- Do NOT repeat artifact content inline. Reference file paths only.
