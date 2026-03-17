# Pipeline Analytics Design

## Problem

The superpowers execution pipeline (executing-plans, subagent-driven-development) logs decisions to the plan doc during execution, but there's no structured way to measure how much the plan changed or analyze decision patterns. Users have no visibility into questions like "how often does the review process change the plan?" or "which tasks generate the most deviations?"

## Goals

- Surface plan changes reliably by preserving an immutable baseline for comparison
- Generate analytics at the end of every execution pipeline automatically
- Keep the first version simple: derive all metrics from diffing the baseline vs final plan doc, with no changes to how the controller operates during execution

## Non-Goals

- Reworking how the execution pipeline operates
- Having the controller accumulate structured data during execution (future version)
- Tracking review loop iterations or implementer status distribution (requires controller changes)
- Real-time decision visibility or approval gates

## Design

### 1. Baseline Creation (changes to existing execution skills)

Both `executing-plans` and `subagent-driven-development` get a new step immediately after loading the plan:

- Copy the plan file to `<plan-name>.baseline.md` in the same directory
- This happens once, before any task execution begins
- The baseline is never modified after creation — it's a frozen snapshot
- The baseline is not cleaned up — it's part of the project record

**Example:** If the plan is `docs/superpowers/plans/2026-03-17-feature-plan.md`, the baseline becomes `docs/superpowers/plans/2026-03-17-feature-plan.baseline.md`.

**Why a file copy instead of git:** A baseline file is self-contained. Any skill, subagent, or analytics process can read two files and diff them without tracking commit SHAs or worrying about git state (worktrees, rebases, amended commits). Simpler and more robust.

### 2. Pipeline Analytics Skill (new skill)

A new skill at `skills/pipeline-analytics/SKILL.md`.

**Inputs:** Path to the plan file. The baseline path is derived by convention (same name with `.baseline.md` suffix).

**What it does:**

1. Reads both the final plan and the baseline
2. Computes metrics by diffing the two
3. Writes a `<plan-name>.analytics.md` file in the same directory

**Metrics (v1):**

| Metric | How it's derived |
|--------|-----------------|
| Lines added/removed/changed | Diff between baseline and final plan |
| Decision count | Count entries in the `## Decisions` section |
| Decisions per task | Group decision entries by their task tag |
| Plan sections modified | Which top-level sections changed (e.g., did task descriptions get edited, or just the Decisions section added?) |

**Output format:**

```markdown
# Pipeline Analytics

**Plan:** <plan-filename>.md
**Date:** <execution-date>

## Summary
- Total decisions logged: N
- Plan lines added: N | removed: N | modified: N
- Plan sections modified: <list>

## Decisions by Task
- Task 1: N decisions
- Task 2: N decisions
...

## Plan Diff
<raw diff between baseline and final>
```

**The analytics skill is invoked directly by the controller in the main session** — not as a subagent. It's reading two files and writing one; no agent dispatch overhead needed.

### 3. Integration into Execution Pipelines

**In `executing-plans`:**

- After Step 1 (Load and Review Plan): copy plan file to `<name>.baseline.md`
- New step after all tasks complete, before finishing-a-development-branch: invoke `superpowers:pipeline-analytics`
- Integration section updated to list `pipeline-analytics` as a required skill

**In `subagent-driven-development`:**

- After reading the plan and extracting tasks: copy plan file to `<name>.baseline.md`
- After the final code review, before finishing-a-development-branch: invoke `superpowers:pipeline-analytics`
- Integration section updated to list `pipeline-analytics` as a required skill

## Future Considerations

These are explicitly out of scope for v1 but could build on this foundation:

- **Controller-accumulated data:** Have the controller track review loop counts, implementer statuses, and model usage during execution, then pass structured data to the analytics skill
- **Cross-run analytics:** Aggregate analytics across multiple executions to identify trends
- **Analytics dashboard:** Visual representation of pipeline metrics over time
