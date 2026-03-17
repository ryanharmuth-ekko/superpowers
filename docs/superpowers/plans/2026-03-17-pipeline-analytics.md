# Pipeline Analytics Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add pipeline analytics to the superpowers execution skills — freeze a baseline copy of the plan before execution, generate an analytics report after execution.

**Architecture:** Three changes: (1) a new `pipeline-analytics` skill that reads a baseline and final plan, computes diff metrics, and writes an analytics markdown file; (2) a one-line baseline copy step added to each execution skill; (3) a one-line analytics invocation step added to each execution skill.

**Tech Stack:** Markdown skill files only — no code, no dependencies.

**Spec:** `docs/superpowers/specs/2026-03-17-pipeline-analytics-design.md`

---

## File Map

- **Create:** `skills/pipeline-analytics/SKILL.md` — the new analytics skill
- **Modify:** `skills/executing-plans/SKILL.md` — add baseline creation + analytics invocation steps
- **Modify:** `skills/subagent-driven-development/SKILL.md` — add baseline creation + analytics invocation steps

---

## Chunk 1: Pipeline Analytics Skill + Execution Skill Integration

### Task 1: Create the pipeline-analytics skill

**Files:**
- Create: `skills/pipeline-analytics/SKILL.md`

- [ ] **Step 1: Create `skills/pipeline-analytics/SKILL.md` with frontmatter and overview**

```markdown
---
name: pipeline-analytics
description: Use at the end of plan execution to generate analytics comparing the baseline plan to the final plan — invoked automatically by executing-plans and subagent-driven-development, not manually
---

# Pipeline Analytics

## Overview

Generate analytics for a completed execution pipeline by comparing the frozen baseline plan to the final plan.

**Announce at start:** "I'm using the pipeline-analytics skill to generate execution analytics."

**This skill is invoked directly in the main session** — not dispatched as a subagent. It reads two files and writes one.

## Inputs

The invoking skill provides the path to the plan file. The baseline path is derived by convention: replace `.md` with `.baseline.md`.

Example: `docs/superpowers/plans/2026-03-17-feature-plan.md` → baseline at `docs/superpowers/plans/2026-03-17-feature-plan.baseline.md`

## The Process

### Step 1: Validate Inputs

1. Confirm the plan file exists and read it
2. Derive the baseline path and confirm it exists
3. If the baseline file does not exist, log a warning and stop. Do not fail the pipeline:
   ```
   ⚠ Baseline file not found at <path>. Skipping analytics generation.
   ```

### Step 2: Generate Diff

Run a unified diff between baseline and final plan:

```bash
diff -u <baseline-path> <plan-path>
```

From the diff output:
- Count lines starting with `+` (excluding `+++` header) → **lines added**
- Count lines starting with `-` (excluding `---` header) → **lines removed**
- Identify which `##` (h2) headings contain diff hunks → **sections modified**

### Step 3: Count Decisions

1. Look for a `## Decisions` section in the final plan
2. If not present, decision count is 0 (this is normal)
3. If present, count non-empty lines between the `## Decisions` heading and the next `##` heading (or end of file)
4. Group entries by task tag prefix — match patterns like `[Task 1]`, `[Task 2]`, `Task 1:`, `Task 2:`, etc. Entries without a recognizable tag go into an "Untagged" group.

### Step 4: Write Analytics File

Write `<plan-name>.analytics.md` in the same directory as the plan. If the file already exists, overwrite it.

Use this format:

```markdown
# Pipeline Analytics

**Plan:** <plan-filename>.md
**Generated:** <YYYY-MM-DD HH:MM>

## Summary
- Total decisions logged: N
- Plan lines added: N | removed: N
- Plan sections modified: <comma-separated list of h2 headings, or "None">

## Decisions by Task
- Task 1: N decisions
- Task 2: N decisions
...
(or "No decisions logged." if count is 0)

## Plan Diff
\`\`\`diff
<raw unified diff output — if exceeding 200 lines, truncate and append "... (diff truncated, N total lines)">
\`\`\`
```

### Step 5: Confirm

Report to the session:
```
Analytics written to <analytics-path>.
```

## Edge Cases

- **No baseline:** Warn and skip (Step 1). Do not fail the pipeline.
- **No changes:** Generate the file with zero metrics and empty diff. This confirms the pipeline ran.
- **No Decisions section:** Report 0 decisions. Normal outcome.
- **Analytics file exists:** Overwrite it.

## Failure Handling

If analytics generation fails for any reason (malformed plan, unexpected format, etc.), log a warning and continue. Analytics must never block the finishing-a-development-branch step.

## Integration

**Called by:**
- **superpowers:executing-plans** — after all tasks complete, before finishing-a-development-branch
- **superpowers:subagent-driven-development** — after final whole-implementation code review, before finishing-a-development-branch
```

**Note:** The analytics output template in Step 4 of the skill uses triple backticks for a diff code fence. When writing the file, these should be literal triple backticks (not escaped).

- [ ] **Step 2: Verify the file was created correctly**

Read `skills/pipeline-analytics/SKILL.md` and confirm:
- Frontmatter has correct `name` and `description`
- All five process steps are present
- Edge cases section matches the spec
- Integration section references both calling skills

- [ ] **Step 3: Commit**

```bash
git add skills/pipeline-analytics/SKILL.md
git commit -m "feat: add pipeline-analytics skill"
```

### Task 2: Add baseline creation and analytics invocation to executing-plans

**Files:**
- Modify: `skills/executing-plans/SKILL.md`

The current structure of `executing-plans/SKILL.md`:
- Step 1: Load and Review Plan
- Step 2: Execute Tasks
- Step 3: Complete Development (invokes finishing-a-development-branch)
- Decision Logging section
- Remember section
- Integration section

- [ ] **Step 1: Add baseline creation to Step 1**

In Step 1 (### Step 1: Load and Review Plan), after the last numbered item "4. If no concerns: Create TodoWrite and proceed", add a new sub-step:

Note: Steps 1 and 2 of this task are independent edits to different parts of Step 1 in the skill file.

```markdown
5. Copy the plan file to `<plan-name>.baseline.md` in the same directory (e.g., `feature-plan.md` → `feature-plan.baseline.md`). This frozen snapshot is used for analytics at the end. If a baseline already exists, overwrite it.
```

- [ ] **Step 2: Add a guard to Step 1**

At the beginning of Step 1, after "1. Read plan file", add:

```markdown
   - If the plan file ends in `.baseline.md` or `.analytics.md`, stop and ask the user for the correct plan file path. These are generated artifacts, not plans.
```

- [ ] **Step 3: Add analytics invocation before Step 3**

Insert a new step between the current Step 2 (Execute Tasks) and Step 3 (Complete Development). Renumber Step 3 → Step 4. The new Step 3:

```markdown
### Step 3: Generate Analytics

After all tasks are complete:
1. Invoke `superpowers:pipeline-analytics` with the plan file path
2. If analytics generation fails or warns, note it and proceed — do not block completion
```

- [ ] **Step 4: Add pipeline-analytics to Integration section**

Add to the Integration section's "Required workflow skills" list:

```markdown
- **superpowers:pipeline-analytics** - Generate analytics after execution completes
```

- [ ] **Step 5: Verify the changes**

Read `skills/executing-plans/SKILL.md` and confirm:
- Baseline copy is in Step 1 (after plan review, before execution)
- Input guard rejects `.baseline.md` and `.analytics.md` files
- New Step 3 invokes pipeline-analytics
- Old Step 3 is now Step 4
- Integration section lists pipeline-analytics

- [ ] **Step 6: Commit**

```bash
git add skills/executing-plans/SKILL.md
git commit -m "feat: add baseline creation and analytics to executing-plans"
```

### Task 3: Add baseline creation and analytics invocation to subagent-driven-development

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

The current structure of `subagent-driven-development/SKILL.md`:
- Overview, When to Use, The Process (flow diagram)
- Model Selection
- Handling Implementer Status
- Decision Logging
- Prompt Templates
- Example Workflow
- Advantages, Cost
- Red Flags
- Integration section

The process flow starts with "Read plan, extract all tasks with full text, note context, create TodoWrite" then loops through implementer → spec review → quality review per task, then "Dispatch final code reviewer subagent for entire implementation" → "Use superpowers:finishing-a-development-branch".

- [ ] **Step 1: Add baseline creation after plan loading**

After the closing of the `digraph process` code block, before the `## Model Selection` heading, add a new section:

```markdown
## Baseline Creation

Immediately after reading the plan and extracting tasks (before dispatching the first implementer):
1. Copy the plan file to `<plan-name>.baseline.md` in the same directory (e.g., `feature-plan.md` → `feature-plan.baseline.md`). This frozen snapshot is used for analytics at the end. If a baseline already exists, overwrite it.
2. If the plan file ends in `.baseline.md` or `.analytics.md`, stop and ask the user for the correct plan file path. These are generated artifacts, not plans.
```

- [ ] **Step 2: Add analytics invocation to the process flow**

In the process flow diagram (the `digraph process` block), add a new node between the final code reviewer and finishing-a-development-branch. Find the edge:

```
    "Dispatch final code reviewer subagent for entire implementation" -> "Use superpowers:finishing-a-development-branch";
```

Replace that single edge with a new node and two edges. Do not modify the existing node declaration for `"Use superpowers:finishing-a-development-branch"` (which has `style=filled fillcolor=lightgreen`) — only change the edge.

```
    "Invoke superpowers:pipeline-analytics" [shape=box];
    "Dispatch final code reviewer subagent for entire implementation" -> "Invoke superpowers:pipeline-analytics";
    "Invoke superpowers:pipeline-analytics" -> "Use superpowers:finishing-a-development-branch";
```

- [ ] **Step 3: Add a Generate Analytics section**

After the `## Decision Logging` section (immediately before `## Prompt Templates`), add:

```markdown
## Generate Analytics

After the final whole-implementation code review subagent completes, before finishing-a-development-branch:
1. Invoke `superpowers:pipeline-analytics` with the plan file path
2. If analytics generation fails or warns, note it and proceed — do not block completion
```

- [ ] **Step 4: Add pipeline-analytics to Integration section**

Add to the Integration section's "Required workflow skills" list:

```markdown
- **superpowers:pipeline-analytics** - Generate analytics after execution completes
```

- [ ] **Step 5: Verify the changes**

Read `skills/subagent-driven-development/SKILL.md` and confirm:
- Baseline Creation section exists after the process flow diagram
- Input guard rejects `.baseline.md` and `.analytics.md` files
- Process flow diagram includes the analytics node between final code review and finishing-branch
- Generate Analytics section exists after Decision Logging
- Integration section lists pipeline-analytics

- [ ] **Step 6: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat: add baseline creation and analytics to subagent-driven-development"
```

### Task 4: Cross-file verification

- [ ] **Step 1: Verify skill name consistency**

Read the frontmatter `name` field in `skills/pipeline-analytics/SKILL.md` and confirm it matches the skill references in:
- `skills/executing-plans/SKILL.md` (Integration section and analytics invocation step)
- `skills/subagent-driven-development/SKILL.md` (Integration section, Generate Analytics section, and process flow diagram)

All references should use `superpowers:pipeline-analytics`.

- [ ] **Step 2: Verify all spec requirements are covered**

Read `docs/superpowers/specs/2026-03-17-pipeline-analytics-design.md` and confirm each design requirement has been implemented:
- Baseline creation in both execution skills
- Input guards against `.baseline.md` / `.analytics.md` files
- Analytics skill with all four metrics (lines added/removed, decision count, decisions per task, sections modified)
- Edge cases handled (no baseline, no changes, no decisions, file exists)
- Failure handling (never blocks pipeline)
- Integration entries in both execution skills
