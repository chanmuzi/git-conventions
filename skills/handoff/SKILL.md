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
| `git-conventions:commit` | git-conventions (self) | Recommend `/commit` for uncommitted changes |
| `git-conventions:pr` | git-conventions (self) | Recommend `/pr` for committed changes without PR |

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

Use this to populate the handoff Context and Delta sections:
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

### Handoff Prompt — With Artifacts

When Layer 4 found a relevant artifact:

```markdown
## Handoff

### Context
{conversation summary — what this session worked on}
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

When no relevant artifact was found (Layer 2 conversation context only):

```markdown
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

- **Context**: Conversation summary (Layer 2) with artifact path reference (if Layer 4 found one). Artifact path supplements — never replaces — the conversation context.
- **Delta**: Thin when artifact is comprehensive (just progress state). Detailed when no artifact captures session work.
- **Next Action**: Include skill name naturally if sentinel detected. Plain task description if no skills detected.
- **Omit empty sections entirely.** If nothing happened beyond creating a spec, omit the Delta section. If git state is clean and irrelevant, omit the Status line.

### Skill Recommendation Mapping

Based on sentinel detection (Layer 1) + context type (Layers 2-4):

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
Conversation analyzed → Relevant artifact found? ──yes──→ Output directly
                              │
                              no
                              │
                              ├── -y flag? ──yes──→ Output directly
                              │
                              no
                              │
                              └── Show draft → "이 내용으로 handoff 하면 될까요?" → [확인] / [수정]
```

- **Relevant artifact found**: Skip confirmation. The handoff is a straightforward reference + thin delta.
- **No relevant artifact, no -y**: Show draft and ask for user confirmation. Prevents incorrect summaries from misleading the next session.
- **-y flag**: Always skip confirmation regardless of artifact presence.

When the user requests modifications after seeing the draft, adjust the handoff accordingly and output the final version.

## Output

Output the final handoff prompt to the terminal. The user can then use `/copy` to copy it for the next session.

**Important:**
- Do NOT create any files. The terminal output IS the deliverable.
- Do NOT execute the recommended next action. Only suggest it.
- Do NOT modify any git state (no commit, push, branch creation). This skill is strictly read-only.
- Do NOT repeat artifact content inline. Reference file paths only.
