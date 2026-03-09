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

Spawn an agent to understand the codebase context relevant to this task. Identify: affected files, dependencies, existing patterns, constraints, and anything that might affect the approach.

Check `.claude/rules/` for project-specific exploration tools (e.g., MCP servers for code navigation). If available, include their loading instructions in the agent's prompt.

### Step 1b: Clarify

After exploration, check whether requirements are clear enough to plan confidently. Use `AskUserQuestion` to resolve any of the following BEFORE proceeding:

- **Ambiguous requirements** — the task could reasonably be interpreted in ways that differ by 2x+ effort
- **Multiple viable approaches** — exploration revealed two or more architectural paths with meaningful trade-offs (e.g., client-side vs server-side, new table vs extending existing)
- **Missing acceptance criteria** — you can't define verification scenarios because the expected behaviour isn't specified
- **Unstated constraints** — the task touches areas with conventions, dependencies, or limitations you can't confirm from the codebase alone

If none of these apply, proceed directly to Step 2.

When asking, be specific about what you found and what decision you need. Present concrete options with trade-offs rather than open-ended questions.

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
  - A **Verification Plan** — concrete, runnable scenarios that prove the feature works from a product/user perspective (see guidance below)

Wait for the agent to return.

#### Verification Plan Guidance

Verification scenarios are NOT restated unit test expectations. They are product-level checks a human (or automated integration test) can execute against the running system.

**Good verification scenarios:**
- "User clicks logout → redirected to login page"
- "Run `curl localhost:3000/api/health` → 200 OK with `{\"status\":\"healthy\"}`"
- "Navigate to /settings → delete account button is visible and functional"
- "Submit form with empty required field → inline validation error appears, form is not submitted"

**Bad verification scenarios (too vague or just restating unit tests):**
- "The logout function works correctly"
- "API returns the right status code"
- "Ensure the component renders"
- "validateEmail() returns false for invalid input"

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
3. **Verification Plan** — The product-level scenarios that prove the feature works end-to-end
4. **Review Findings** — Concerns, gaps, or suggestions from the reviewer (or "none" if clean)
5. **Next** — "Approve this plan, then run `/orc:tasks` to create tasks."

## Rules

- NEVER skip the Explore step — plans without codebase context are guesses
- NEVER skip the Review step — unreviewed plans have blind spots
- NEVER begin execution — this skill produces a plan, not code
- NEVER use `EnterPlanMode` / `ExitPlanMode` — they conflict with this flow
- If the reviewer raises critical issues, revise before presenting — don't present a plan you know is flawed
