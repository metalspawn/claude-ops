---
name: tasks
description: "Create detailed tasks from an approved plan with file references and acceptance criteria. Run after /orc:plan approval, before /orc:execute."
---

# /orc:tasks — Create Tasks from Approved Plan

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

1. Read the approved plan in full
2. Break it into discrete tasks — one per logical unit of work (not one per file, not one monolithic task)
3. Create all tasks via `TaskCreate`
4. Set up dependencies using `addBlockedBy` / `addBlocks` where the plan defines ordering constraints
5. Present the created task list to the user for confirmation before execution begins

## Rules

- Every task MUST have acceptance criteria — no exceptions
- Reference specific file paths from the plan, not vague descriptions
- Tasks should be ordered so that foundational work (types, utilities, API layers) comes before dependent work (UI components, integration)
- Do NOT begin executing tasks — only create them. Execution starts with `/orc:execute` after the user confirms.
