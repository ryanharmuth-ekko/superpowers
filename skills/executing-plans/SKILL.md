---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

## Overview

Load plan, review critically, execute all tasks, report when complete.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

**Note:** Tell your human partner that Superpowers works much better with access to subagents. The quality of its work will be significantly higher if run on a platform with subagent support (such as Claude Code or Codex). If subagents are available, use superpowers:subagent-driven-development instead of this skill.

## The Process

### Step 1: Load and Review Plan
1. Read plan file
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed

### Step 2: Execute Tasks

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Log any non-trivial decisions to the plan doc (see Decision Logging below)
5. Mark as completed

### Step 3: Complete Development

After all tasks complete and verified:
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
- Follow that skill to verify tests, present options, execute choice

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly

**Ask for clarification rather than guessing.**

## When to Revisit Earlier Steps

**Return to Review (Step 1) when:**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking

**Don't force through blockers** - stop and ask.

## Decision Logging

After completing each task, log any non-trivial decisions to a `## Decisions` section at the **top** of the plan doc (right after the title, before everything else). This ensures the human can open the plan and immediately see what changed.

**Log when you:**
- Deviate from the plan — "Plan said X, did Y because Z"
- Resolve ambiguity — "Spec didn't cover this, chose to..."
- Make a tradeoff — "Could have done A or B, went with A because..."
- Discover something that changes the approach — "Found existing code handles X, so..."
- Skip or defer something — "Deferred X because Y"

**Don't log:** Routine implementation, things that exactly follow the plan, formatting/naming choices.

**Format:** Reverse chronological (newest first). Each entry is one line: task tag, what was decided, brief why.

```markdown
## Decisions

- **Task 3:** Used polling instead of webhooks — target API doesn't support webhooks
- **Task 2:** Added migration step not in plan — existing data had nulls that would break new constraint
```

If the section doesn't exist yet, create it after the first heading. Insert new entries at the top. This is not a gate — continue working. It's a log.

## Remember
- Review plan critically first
- Follow plan steps exactly
- Don't skip verifications
- Reference skills when plan says to
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
