---
name: execute
description: "Execute tasks through the worker, review, validate, and commit pipeline. Run after /orc:tasks, or directly with a task description for simple work."
---

# /orc:execute — Execute Tasks

## Input

$ARGUMENTS

## Precondition

Tasks MUST already exist (created via `/orc:tasks` or manually). If no tasks exist, stop and tell the user to run `/orc:tasks` first.

**Exception — direct execution:** If `$ARGUMENTS` contains a concrete, self-contained task description (not a reference to a plan), create a single task with acceptance criteria inferred from the description, then proceed.

---

## Execution Loop

For each task, follow these steps in exact order. Do NOT skip ahead. Each step has a gate.

### Step 1: Pick up the task
- `TaskGet` to read the full description and acceptance criteria.
- Confirm all `blockedBy` tasks are complete. If blocked, skip to the next unblocked task.
- `TaskUpdate(status='in_progress')`.

### Verification Tasks

If the task has `metadata.type === "verification"`, skip the main loop (Step 2 Implement, Step 3 Review + Validate, Step 4 Commit) and follow this path instead:

- Spawn the `validator` agent with the verification scenarios from the task description.
- **PASS** → proceed to Step 5 (mark complete).
- **FAIL** → report failing scenarios. Spawn `worker` to fix the failing areas. Re-run validator. Repeat until PASS.

### Step 2: Implement — delegate to `worker`
- Spawn the `worker` agent with the task description, acceptance criteria, and file paths.
- NEVER edit implementation files yourself. NEVER substitute with direct tool calls.
- Wait for the worker to return.

⛔ **STOP. The worker is done. Do NOT commit. Three mandatory gates remain.**

### Step 3: Review + Validate — spawn ALL THREE in parallel
Spawn these agents simultaneously via parallel Task tool calls. **None is optional.**

**`code-reviewer`:**
- Scope: **only files changed by the worker** — the diff, not the repo.
- Checks against: project CLAUDE.md conventions, framework idioms, separation of concerns, naming and file placement.
- NEVER skip because "the code looks fine" — convention drift is invisible without review.

**`semantic-reviewer`:**
- Scope: **only files changed by the worker**.
- Checks naming clarity, comment accuracy, readability.
- Binary PASS/FAIL — if it's worth mentioning, it's worth fixing.

**`validator`:**
- Run tests (existing + any new ones), linting, type checking.
- Confirm acceptance criteria from the task are met.
- Return explicit pass/fail.

Wait for **ALL THREE** to return before proceeding.

**Handling results:**
- **All PASS:** proceed to Step 4.
- **Any FAIL:** delegate fixes to `worker`, then re-run **ALL THREE** agents in parallel. A fix could introduce new issues in any domain. Repeat until no FAILs.

### Step 4: Commit
You may ONLY reach this step with:
- ✅ `code-reviewer` PASS (this cycle)
- ✅ `semantic-reviewer` PASS (this cycle)
- ✅ `validator` PASS (this cycle)
- ✅ No unrelated changes staged

Follow [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): description`
- Infer type: `feat`, `fix`, `chore`, `refactor`, `test`, `docs`
- Imperative mood
- Logically grouped changes — not everything in one commit, not every line separately

### Step 5: Complete
- `TaskUpdate(status='completed')`.
- Check `TaskList` for the next unblocked task. Return to Step 1.

---

## Rules

- NEVER execute if no tasks exist (unless direct execution via arguments)
- NEVER edit implementation files yourself — delegate to `worker`
- NEVER run tests yourself — delegate to `validator`
- NEVER skip any of the three review gates
- NEVER commit without all three gates passing in the current cycle
- NEVER re-run only the failed gate after a fix — ALL THREE must re-run
- Verification tasks skip worker/reviewers/commit — validator only. Verification FAIL blocks completion.
- If blocked by human-required input (credentials, decisions, access), stop and report
