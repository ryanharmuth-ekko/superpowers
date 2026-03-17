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
   - If the plan file ends in `.baseline.md` or `.analytics.md`, stop and ask the user for the correct plan file path. These are generated artifacts, not plans.
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed
5. Copy the plan file to `<plan-name>.baseline.md` in the same directory (e.g., `feature-plan.md` → `feature-plan.baseline.md`). This frozen snapshot is used for analytics at the end. If a baseline already exists, overwrite it.

### Step 2: Execute Tasks

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Log any non-trivial decisions to the plan doc (see Decision Logging below)
5. Mark as completed

### Step 3: Generate Analytics

After all tasks are complete:
1. Invoke `superpowers:pipeline-analytics` with the plan file path
2. If analytics generation fails or warns, note it and proceed — do not block completion

### Step 4: Complete Development

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

After completing each task, log non-trivial decisions to a `## Decisions` section at the **top** of the plan doc (after the title, before everything else). Reverse chronological — newest first. One line per entry: task tag, what was decided, brief why.

Log: deviations from plan, ambiguity resolution, tradeoffs, discoveries that changed the approach. Don't log routine/mechanical work.

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
- **superpowers:pipeline-analytics** - Generate analytics after execution completes
