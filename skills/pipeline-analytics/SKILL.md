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
```diff
<raw unified diff output — if exceeding 200 lines, truncate and append "... (diff truncated, N total lines)">
```
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
