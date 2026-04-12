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

## Convention Detection

Run these checks once before entering the execution loop.

### Step 0: Detect commit conventions

Check these sources in priority order. Use the first match found:

1. **Project CLAUDE.md** — Read the project's CLAUDE.md. Look for a section with heading containing "Commit" (e.g., "Commit Messages", "Commit Convention", "Commits"). If found, follow that convention exactly. Skip remaining checks.

2. **Commitlint config** — Check for these files at the repo root (in this order):
   - `.commitlintrc.json`
   - `.commitlintrc.yml` / `.commitlintrc.yaml`
   - `.commitlintrc.js` / `.commitlintrc.cjs` / `.commitlintrc.mjs`
   - `commitlint.config.js` / `commitlint.config.cjs` / `commitlint.config.mjs` / `commitlint.config.ts`

   If found and the config is a static JSON/YAML file or a JS/TS file with a plain object export:
   - `extends: ["@commitlint/config-conventional"]` or `extends: ["@commitlint/config-angular"]` → confirms the preset format. Apply any custom `rules` (e.g., `type-enum`, `scope-enum`, `subject-case`).
   - Other static configs → extract the format rules and apply them.
   - Complex configs (dynamic rules, conditional logic, plugin imports) → fall back to default.

3. **Default** — [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): description`

**Precedence rule:** Detected conventions replace **format rules** (message structure, type list, scope constraints). They do NOT replace **process rules** (logical grouping, gate requirements, no unrelated changes staged).

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

### Step 3: Validate
Spawn the `validator` agent:
- Run tests (existing + any new ones) and type checking.
- Return explicit pass/fail.

**Handling results:**
- **PASS:** proceed to Step 4.
- **FAIL:** delegate fixes to `worker`, then re-run `validator`. Repeat until PASS.

### Step 4: Commit
You may ONLY reach this step with:
- ✅ `validator` PASS (this cycle)
- ✅ No unrelated changes staged

Apply the commit convention detected in Step 0.
- Imperative mood (unless the detected convention specifies otherwise)
- Logically grouped changes — not everything in one commit, not every line separately

### Step 5: Complete
- `TaskUpdate(status='completed')`.
- Check `TaskList` for the next unblocked task. Return to Step 1.

---

## Rules

- NEVER execute if no tasks exist (unless direct execution via arguments)
- NEVER edit implementation files yourself — delegate to `worker`
- NEVER run tests yourself — delegate to `validator`
- NEVER skip the validator gate
- NEVER commit without validator PASS in the current cycle
- Verification tasks skip worker/reviewers/commit — validator only. Verification FAIL blocks completion.
- ALWAYS detect commit conventions before committing — Step 0 is mandatory
- If blocked by human-required input (credentials, decisions, access), stop and report
