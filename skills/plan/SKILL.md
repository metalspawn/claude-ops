---
name: plan
description: "Develop a structured implementation plan through codebase exploration, planning, and critical review. Use for non-trivial work: multi-file changes, architectural decisions, new features, or ambiguous requirements."
---

# /orc:plan — Develop an Implementation Plan

## Input

$ARGUMENTS

## Process

Follow these steps in exact order. Each step produces output that feeds the next.

### Step 1: Explore

Spawn the `Explore` agent (thoroughness: medium) to understand the codebase context.

Provide:
- The task description above
- Request to identify: affected files, dependencies, existing patterns, constraints
- Request to surface anything that might affect the approach

If the task is ambiguous (2x+ effort difference between interpretations), ask clarifying questions BEFORE proceeding.

Wait for the agent to return.

### Step 2: Plan

Spawn the `Plan` agent to produce a structured implementation plan.

Provide:
- The original task description
- The exploration findings from Step 1
- Request a plan that defines:
  - What changes, where, in what order
  - Verification criteria for each step
  - Dependencies between steps
  - Risks or edge cases

Wait for the agent to return.

### Step 3: Review

Spawn the `plan-reviewer` agent to critique the plan.

Provide:
- The full plan from Step 2
- The exploration context from Step 1
- Request it to identify: flaws, edge cases, gaps, missing dependencies, over-engineering

Wait for the agent to return.

If the reviewer raises critical issues, revise the plan (return to Step 2 with review feedback) before presenting.

### Step 4: Present for Approval

Present the plan and review findings. Format as:

1. **Summary** — One-line description of what the plan achieves
2. **Steps** — The structured plan with discrete steps, file paths, and verification criteria
3. **Review Findings** — Concerns, gaps, or suggestions from the reviewer (or "none" if clean)
4. **Next** — "Approve this plan, then run `/orc:tasks` to create tasks."

## Rules

- NEVER skip the Explore step — plans without codebase context are guesses
- NEVER skip the Review step — unreviewed plans have blind spots
- NEVER begin execution — this skill produces a plan, not code
- NEVER use `EnterPlanMode` / `ExitPlanMode` — they conflict with this flow
- If the reviewer raises critical issues, revise before presenting — don't present a plan you know is flawed
