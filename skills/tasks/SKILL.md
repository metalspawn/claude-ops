---
name: tasks
description: "Create detailed tasks from an approved plan with file references and acceptance criteria. Run after /orc:plan approval, before /orc:execute."
---

# /orc:tasks — Create Tasks from Approved Plan

## Input

$ARGUMENTS

Read the approved plan and create tasks using `TaskCreate`. Each task MUST include:

## Required Fields

- **subject** — Clear, imperative title (e.g., "Add authentication middleware")
- **activeForm** — Present continuous form for the spinner (e.g., "Adding authentication middleware")
- **description** — Must contain ALL of the following:
  1. **What to change** — Specific description of the implementation work
  2. **Where** — File paths that will be created or modified, referencing existing files by their full path
  3. **Why** — Context from the plan explaining the purpose of this change
  4. **Acceptance criteria** — Explicit, verifiable conditions that define "done". Each criterion should be a concrete check, not a vague statement. Examples:
     - "The `useAuth` hook returns `{ user, isLoading, error }` matching the `AuthState` type"
     - "Clicking the logout button calls `authAPI.logout()` and redirects to `/login`"
     - "Tests pass for both authenticated and unauthenticated states"
     - NOT "Authentication works correctly" (too vague)
     - NOT "Code is clean" (not verifiable)

## Process

1. **Branch setup** — Invoke `/orc:branch` via the `Skill` tool, passing `$ARGUMENTS`. If already on a feature branch, this is a no-op.
2. Read the approved plan in full
3. Break it into discrete tasks — one per logical unit of work (not one per file, not one monolithic task)
4. Create all tasks via `TaskCreate`
5. Set up dependencies using `addBlockedBy` / `addBlocks` where the plan defines ordering constraints
6. **Create verification task** — Create a final task from the plan's Verification Plan section:
   - Subject reflects the specific feature (e.g., "Verify authentication behaviour")
   - `metadata: { type: "verification" }`
   - `blockedBy` all implementation tasks
   - Description contains the verification scenarios from the plan verbatim — these ARE the acceptance criteria (do not add separate acceptance criteria)
   - activeForm: present continuous matching the subject (e.g., "Verifying authentication behaviour")
7. Present the created task list to the user for confirmation before execution begins

## Rules

- Every task MUST have acceptance criteria — no exceptions
- Reference specific file paths from the plan, not vague descriptions
- Tasks should be ordered so that foundational work (types, utilities, API layers) comes before dependent work (UI components, integration)
- The verification task MUST be the last task created, blocked by all implementation tasks
- Do NOT begin executing tasks — only create them. Execution starts with `/orc:execute` after the user confirms.
