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

**Guards:**

- Execution skills must never accept a `.baseline.md` or `.analytics.md` file as a plan input.
- If a baseline file already exists (re-run scenario), overwrite it. The previous baseline is no longer meaningful since the plan itself may have been updated between runs. Note: this means resuming an interrupted pipeline will reset the baseline — acceptable for v1.
- If execution is interrupted before analytics generation, the baseline file remains as a useful artifact for manual comparison. No cleanup is needed.

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
| Lines added / removed | Use `diff -u` (unified diff) between baseline and final plan. Count `+` and `-` lines, excluding diff headers. |
| Decision count | Count entries in the `## Decisions` section. Zero is a valid result (all tasks were mechanical). |
| Decisions per task | Group decision entries by task tag prefix (e.g., `[Task 1]`). Best-effort — entries without a recognizable tag go into an "Untagged" group. |
| Plan sections modified | Which `##` (h2) headings have diff hunks beneath them. Identifies whether changes were in task descriptions, the Decisions section, or elsewhere. |

**Output format:**

```markdown
# Pipeline Analytics

**Plan:** <plan-filename>.md
**Generated:** <date and time when analytics were generated>

## Summary
- Total decisions logged: N
- Plan lines added: N | removed: N
- Plan sections modified: <list>

## Decisions by Task
- Task 1: N decisions
- Task 2: N decisions
...

## Plan Diff
<raw unified diff between baseline and final — if exceeding 200 lines, truncate and note "diff truncated">
```

**The analytics skill is invoked directly in the main session** — not dispatched as a subagent. It's reading two files and writing one; no agent dispatch overhead needed.

**Edge cases:**

- If the baseline file does not exist, log a warning and skip analytics generation. Do not fail the pipeline.
- If the plan has no `## Decisions` section, report zero decisions. This is a normal outcome.
- If the baseline and final plan are identical (no changes), still generate the analytics file with zero metrics and an empty diff. This confirms the pipeline ran and nothing changed.
- If an analytics file already exists (re-run), overwrite it.

### 3. Integration into Execution Pipelines

**In `executing-plans`:**

- After Step 1 (Load and Review Plan): copy plan file to `<name>.baseline.md`
- New step after all tasks complete, before finishing-a-development-branch: invoke `superpowers:pipeline-analytics`
- Add to Integration section: `- **superpowers:pipeline-analytics** - Generate analytics after execution completes`

**In `subagent-driven-development`:**

- After reading the plan and extracting tasks: copy plan file to `<name>.baseline.md`
- After the final whole-implementation code review subagent completes, before finishing-a-development-branch: invoke `superpowers:pipeline-analytics`
- Add to Integration section: `- **superpowers:pipeline-analytics** - Generate analytics after execution completes`

## Failure Handling

Analytics generation failures (malformed plan, unexpected format, etc.) must be logged as a warning but must not block the `finishing-a-development-branch` step. The analytics are informational — they should never prevent work from being completed.

## Future Considerations

These are explicitly out of scope for v1 but could build on this foundation:

- **Controller-accumulated data:** Have the controller track review loop counts, implementer statuses, and model usage during execution, then pass structured data to the analytics skill
- **Cross-run analytics:** Aggregate analytics across multiple executions to identify trends
- **Analytics dashboard:** Visual representation of pipeline metrics over time
