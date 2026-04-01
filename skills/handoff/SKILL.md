---
name: handoff
description: >-
  Generate a copy-ready handoff prompt for transferring work context to a new session.
  TRIGGER when: user asks to hand off work, create a handoff prompt, transfer context, wrap up session, prepare for next session (e.g., "handoff 해줘", "다음 세션으로 넘겨줘", "작업 이관해줘", "handoff prompt 만들어줘").
  DO NOT TRIGGER when: user is committing, creating PRs, reviewing code, or performing other git operations without handoff intent.
version: "1.0.0"
allowed-tools: Bash(git *), Read, Glob, Grep
---

## Core Principle

> **Never repeat artifact content.** If a spec, plan, or document exists, reference its path. The handoff prompt contains only: (1) pointers to existing artifacts, (2) session delta that exists nowhere else, (3) a single concrete next action with skill recommendation.

## Parse Arguments

Parse `$ARGUMENTS` to determine the handoff mode.

| Argument | Type | Description |
|----------|------|-------------|
| `topic` | Optional string | Focus filter — handoff concentrates on this topic only (e.g., `/handoff auth 리팩토링`) |
| `-y` | Optional flag | Skip confirmation — output handoff prompt directly without draft review |

If `$ARGUMENTS` contains `-y`, extract it as the skip-confirmation flag. The remaining text (if any) is the topic filter.

If `$ARGUMENTS` is empty, proceed with auto-detection and default confirmation flow.

## Detection Cascade

Run the following detection layers in priority order. Each layer enriches the handoff context; later layers run only if earlier layers did not provide sufficient context.

---

**Layer 0 — Topic Override**

If a topic was provided in `$ARGUMENTS`, use it to scope the handoff. The topic guides which parts of the session context to include. Continue to subsequent layers for context detection, but filter all results through this topic.

---

**Layer 1 — Sentinel Skill Detection (Zero-cost)**

Check if these sentinel skill names exist in the current session's available skills. This is zero-cost: no Bash, no filesystem reads — check the session's context window for skill names.

| Sentinel | Plugin | What it enables |
|----------|--------|-----------------|
| `oh-my-claudecode:autopilot` | OMC | Recommend `/autopilot` or `/ralph` for spec/plan-based execution |
| `git-conventions:commit` | git-conventions (self) | Recommend `/commit` for uncommitted changes |
| `git-conventions:pr` | git-conventions (self) | Recommend `/pr` for committed changes without PR |

Store detected sentinels for use in the skill recommendation step.

---

**Layer 2 — Artifact Detection**

Search for existing structured artifacts using the Glob tool:

```
Glob: .omc/specs/*.md
Glob: .omc/plans/*.md
```

Combine results from both patterns.

If artifacts are found:
- Use the topic filter (if provided) and the current conversation context to determine which artifact is most relevant.
- If multiple artifacts exist and it is ambiguous which one is relevant, list candidates and ask the user to confirm which to hand off. This is **artifact selection confirmation** — separate from the draft-summary confirmation in Layer 4.
- You may read the first few lines of the selected artifact **only to understand what it is about and to choose the right artifact**. Do **not** copy or quote any artifact content into the handoff; in the Context section, reference the artifact **by its path only**.

Once the relevant artifact has been selected (or was unambiguous from the start), this is the **artifact-present** path. Skip draft-summary confirmation — generate the final handoff prompt directly.

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

Use this to populate the handoff Context and Delta sections:
- Current branch name
- Uncommitted changes (count and nature)
- Recent commits (what was done this session)
- Stashed work (if any)

---

**Layer 4 — Conversation Context Fallback**

If no artifacts were found (Layer 2 empty) and git state alone is insufficient to describe the session work:
- Analyze the current conversation to extract key decisions, conclusions, and agreed direction.
- This becomes the primary content for the handoff prompt.
- **This layer triggers the confirmation flow** (unless `-y` flag is set), because conversation-derived summaries need user validation before handoff.

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

Everything below the divider is the actual handoff prompt that the user copies to the next session.

### Handoff Prompt — With Artifacts

When Layer 2 found a relevant artifact:

```
## Handoff

### Context
- Spec: `{artifact_path}`
- Branch: `{current_branch}`
- Status: {brief git state — clean / N files modified / stash exists}

### Delta
- {session progress — what was completed since the artifact was created}
- {discoveries — what was found that is not captured in the artifact}
- {blockers — what is blocking next steps, if any}

### Next Action
{skill recommendation woven naturally into the action description}
```

Example Next Action with OMC detected:
```
/autopilot으로 `.omc/specs/deep-interview-code-review.md` 참고하여 Step 4 cross-validation부터 진행
```

Example Next Action without OMC:
```
`.omc/specs/deep-interview-code-review.md` 참고하여 Step 4 cross-validation부터 진행
```

### Handoff Prompt — Without Artifacts

When Layer 4 fallback was used:

```
## Handoff

### Context
{conversation summary — topic, key decisions, agreed direction}
- Branch: `{current_branch}`
- Status: {brief git state}

### Delta
- {key decisions made this session}
- {rejected alternatives and why}
- {open questions remaining}

### Next Action
{specific task description with concrete starting point}
```

### Adaptive Section Rules

- **Context**: Artifact path reference (if exists) OR conversation summary (if not). Never both.
- **Delta**: Thin when artifact is comprehensive (just progress state). Detailed when no artifact captures session work.
- **Next Action**: Include skill name naturally if sentinel detected. Plain task description if no skills detected.
- **Omit empty sections entirely.** If nothing happened beyond creating a spec, omit the Delta section. If git state is clean and irrelevant, omit the Status line.

### Skill Recommendation Mapping

Based on sentinel detection (Layer 1) + context type (Layers 2-3):

| Context | OMC Available | No OMC | Strategy |
|---------|---------------|--------|----------|
| Spec exists (`.omc/specs/`) | `/autopilot` or `/ralph` | "spec 참고하여 작업 진행" | Spec-driven execution |
| Plan exists (`.omc/plans/`) | `/autopilot` | "plan 참고하여 작업 진행" | Plan-driven execution |
| Code changes (uncommitted) | `/commit` | `/commit` | Stage and commit first |
| Committed, no PR | `/pr` | `/pr` | Create PR |
| Conversation only | Context-dependent | Plain task description | Varies by topic |

**Important:** Weave the skill name into the Next Action naturally:
- Good: `/autopilot으로 spec 참고하여 Step 4부터 진행`
- Bad: `Step 4부터 진행 (추천: /autopilot)`

## Confirmation Flow

```
Has artifact? ──yes──→ Output directly
     │
     no
     │
     ├── -y flag? ──yes──→ Output directly
     │
     no
     │
     └── Show draft → "이 내용으로 handoff 하면 될까요?" → [확인] / [수정]
```

- **Artifact present**: Skip confirmation. The handoff is a straightforward reference + thin delta.
- **No artifact, no -y**: Show draft and ask for user confirmation. Prevents incorrect summaries from misleading the next session.
- **-y flag**: Always skip confirmation regardless of artifact presence.

When the user requests modifications after seeing the draft, adjust the handoff accordingly and output the final version.

## Output

Output the final handoff prompt to the terminal. The user can then use `/copy` to copy it for the next session.

**Important:**
- Do NOT create any files. The terminal output IS the deliverable.
- Do NOT execute the recommended next action. Only suggest it.
- Do NOT modify any git state (no commit, push, branch creation). This skill is strictly read-only.
- Do NOT repeat artifact content inline. Reference file paths only.
