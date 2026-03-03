---
name: branch
description: "Create a feature branch from the current branch. Invoked directly or from /orc:tasks. No-op if already on a feature branch."
---

# /orc:branch — Branch Setup

## Input

$ARGUMENTS

## Definitions

**Protected branches:** `main`, `master`, `develop`. These are the only branches treated as protected by this skill.

## Process

The orchestrator runs this directly via Bash tool — no agent delegation.

### Step 1: Detect current branch

Run `git branch --show-current`.

- If the current branch is NOT a protected branch → report "On branch: `<name>`" and **STOP**. This is a no-op.
- If on a protected branch → proceed to Step 2.

### Step 2: Determine branch name

#### 2a: Check project conventions

Read the project's CLAUDE.md and look for a "Branch" or "Branch Naming" section. If found, follow that convention exactly and skip 2b.

#### 2b: Default convention

If no project convention exists, derive the branch name from `$ARGUMENTS`:

**Ticket:** Parse `$ARGUMENTS` for a ticket reference matching `[A-Z]+-\d+`, OR look for `--ticket <ID>`. Extract if found.

**Type:** Infer from keywords in arguments:

| Keyword(s) | Type |
|---|---|
| `fix`, `bug` | `fix` |
| `feat`, `feature`, `add` | `feat` |
| `refactor` | `refactor` |
| `chore` | `chore` |
| `docs`, `documentation` | `docs` |
| `test` | `test` |

Default if no keyword matches: `feat`

**Slug:** Remaining words (after removing the ticket reference and the matched keyword that triggered the type — e.g., remove `bug` not `fix` when `bug` was the input), kebab-case, max 5 words.

**Format:**
- With ticket: `<type>/<ticket>-<slug>`
- Without ticket: `<type>/<slug>`

### Step 3: Create branch

1. Check if the branch name already exists: `git branch --list <name>`
   - If it exists → **STOP** and ask the user what to do
2. Run `git checkout -b <branch-name>`
3. Report: "Created branch: `<name>`"

---

## Edge Cases

- **No `$ARGUMENTS` and no description available** → prompt the user for a branch description. Do NOT guess.
- **Branch already exists** → STOP, ask the user whether to check it out, rename, or abort.
- **Already on a non-protected branch** → report and STOP. No branch creation.

---

## Examples

| Input | Result |
|---|---|
| `/orc:branch --ticket PROJ-123 Add dark mode` | `feat/PROJ-123-add-dark-mode` |
| `/orc:branch fix login session expiry AUTH-45` | `fix/AUTH-45-login-session-expiry` |
| `/orc:branch Add user search` | `feat/add-user-search` |
| `/orc:branch refactor database connection pool` | `refactor/database-connection-pool` |
| `/orc:branch docs update API reference` | `docs/update-api-reference` |

---

## Rules

- NEVER create a branch if already on a non-protected branch — report and STOP
- NEVER force-create a branch that already exists
- ALWAYS check project CLAUDE.md for branch naming conventions before using defaults
- ALWAYS read `$ARGUMENTS` — if empty and no context available, prompt the user. Do NOT invent a branch name.
- NEVER delegate this to an agent — the orchestrator runs it directly via Bash tool
