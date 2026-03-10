---
name: decompose
description: "Break a feature into PR-sized, independently shippable steps. Use before /orc:plan when work spans multiple PRs — architectural changes, large features, or multi-PR rollouts."
---

# /orc:decompose — Decompose a Feature into Shippable Steps

## Input

$ARGUMENTS

## Process

Follow these steps in exact order. Each step produces output that feeds the next.

### Step 1: Explore

Spawn an agent to thoroughly understand the codebase context. Identify: affected systems, module boundaries, dependencies between components, existing patterns, integration points. Surface: what can change independently, what has coupling that forces ordering, what existing tests or contracts constrain the work.

### Step 1b: Clarify

After exploration, check whether the feature is well-enough defined to decompose confidently. Use `AskUserQuestion` to resolve any of the following BEFORE proceeding:

- **Ambiguous scope** — the feature could reasonably be interpreted in ways that differ by 2x+ effort (e.g., "add auth" could mean OAuth, magic links, or both)
- **Unclear boundaries** — exploration revealed coupling or shared concerns where the natural split point isn't obvious (e.g., should the API and UI ship together or separately?)
- **Missing context** — the feature references systems, constraints, or requirements you can't verify from the codebase alone

If none of these apply, proceed directly to Step 2.

When asking, be specific about what you found and what decision you need. Present concrete options where possible rather than open-ended questions.

### Step 2: Decompose

Using the exploration findings, break the feature into PR-sized steps.

For each step, define exactly four fields:
- **Description** — What this step accomplishes (one or two sentences)
- **Scope** — Which areas of the codebase it touches (modules, layers, services — NOT file paths)
- **Depends on** — Which prior steps must be merged first (or "None" for the first step)
- **Shippable Artefact** — What the PR delivers that is independently valuable (a working capability, not "part 1 of N")

Order steps so that:
- Each step builds on the previous without leaving the codebase broken
- Earlier steps lay groundwork that later steps depend on
- The highest-value or highest-risk work comes first where dependencies allow

If the feature is small enough for a single PR, stop here. Tell the user: "This feature fits in a single PR. Run `/orc:plan <feature description>` directly instead."

### Step 3: Present for Approval

Present the decomposition to the user. Format as:

1. **Summary** — One-line description of what the feature delivers and how many steps it requires
2. **Steps** — Ordered list with all four fields per step:
   ```
   ### Step 1: <short title>
   - **Description:** ...
   - **Scope:** ...
   - **Depends on:** None
   - **Shippable Artefact:** ...

   ### Step 2: <short title>
   - **Description:** ...
   - **Scope:** ...
   - **Depends on:** Step 1
   - **Shippable Artefact:** ...
   ```
3. **Next** — "Approve this decomposition, then run `/orc:plan <Step 1 description>` to begin."

## Rules

- NEVER skip the Explore step — decomposition without codebase context produces artificial boundaries
- NEVER produce implementation-level detail (file paths, specific code changes) — that is `/orc:plan`'s job
- NEVER begin execution — this skill produces a decomposition, not code
- NEVER invoke the plan-reviewer — the user reviews decompositions, agents review implementation plans
- NEVER use `EnterPlanMode` / `ExitPlanMode` — they conflict with this flow
- Each step MUST be independently shippable — tests pass, nothing broken, no "part 1 of 2" that leaves the codebase in a broken state
- **Bias toward smaller steps.** Each step should be small enough to review efficiently — a reviewer should be able to hold the entire change in their head. Several small PRs beats one big one; the cost of splitting is low (agents handle the git mechanics) while the cost of a PR too large to review effectively is high (superficial reviews, missed issues, merge pain). When in doubt, ask the user whether to split further.
- Steps MUST be ordered so dependencies are respected — no step may depend on a later step
- If the feature fits in a single PR, say so and recommend `/orc:plan` directly — do not over-decompose
