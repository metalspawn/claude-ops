---
name: ship
description: "Push current branch, create or update a PR, run self-review, and triage findings. Run after /orc:execute completes."
---

# /orc:ship — Ship and Review

## Input

$ARGUMENTS

---

## Process

Follow these steps in exact order. Do NOT skip ahead. Steps 1–3 are run directly by the orchestrator via Bash/tool calls.

### Step 1: Preflight

Run these checks before doing anything else.

**Branch check:**
- Get current branch: `git branch --show-current`
- STOP immediately if branch is `main`, `master`, or `develop`. Tell the user: "Cannot ship from a protected branch. Switch to a feature branch first."

**Commits ahead check:**
- Run `git log origin/main..HEAD --oneline`. If `origin/main` does not exist, try `origin/master`.
- STOP if no commits are ahead. Tell the user: "No commits ahead of base branch — nothing to ship."

**Uncommitted changes check:**
- Run `git status --porcelain`.
- If output is non-empty: show advisory warning — "Uncommitted changes detected — these may belong to another branch." Do NOT fail.

**Task status check:**
- Run `TaskList`.
- If incomplete tasks exist (status `pending` or `in_progress`): show advisory warning — "Incomplete tasks remain." Do NOT fail.

### Step 2: Push

Push the current branch to the remote.

- Check if branch tracks a remote: `git rev-parse --abbrev-ref @{upstream}`
- If tracking exists: `git push`
- If no tracking (command fails): `git push -u origin $(git branch --show-current)`

### Step 3: Create or update PR

Invoke `/orc:pr` via the Skill tool: `skill: "orc:pr"`

The pr skill handles:
- Checking for an existing PR
- Detecting project PR conventions (title format, body template)
- Creating the PR with detected conventions or sensible defaults

If `/orc:pr` reports an existing PR, note the URL and proceed to Step 4.

### Step 4: Self-review

This step is mandatory. NEVER skip it.

- Invoke the built-in `/review` skill via the `Skill` tool: `skill: "review"`
- The review findings are returned directly in the conversation — they are NOT posted to GitHub.
- Parse the returned findings into structured items. Each item MUST have:
  - **file** — the file path
  - **line** — the line number (or range)
  - **description** — what the finding is

### Step 5: Triage findings

If no findings from the review, skip to Step 6 with "no findings".

**Classify each finding into one of three severity tiers:**

- **Must Address** — bugs, security issues, logic errors, data loss risks
- **Should Consider** — code quality, performance, maintainability concerns
- **Nitpick** — style, naming preferences, minor readability

**Present findings as a numbered list grouped by category:**

```
### Must Address
1. [file:line] Description

### Should Consider
2. [file:line] Description

### Nitpick
3. [file:line] Description
```

**You MUST stop here and ask the user which findings to address.** Tell them they can pick numbers (e.g., "1, 3"), "all", or "none". Do NOT proceed to Step 6 until the user responds.

### Step 6: Next steps

This step runs ONLY after the user has responded to the Step 5 prompt. Based on user selection:

- **Findings selected (non-trivial — multi-file or architectural):**
  Summarise the selected findings as a brief. Tell the user:
  "Run `/orc:plan` to plan the changes, then `/orc:tasks` → `/orc:execute` → `/orc:ship`."

- **Findings selected (trivial — single-file, obvious fix):**
  Summarise the selected findings. Tell the user:
  "Run `/orc:execute <description>` to address directly, then `/orc:ship`."

- **No findings or "none" selected:**
  "PR is ready for human review." Report the PR URL.

---

## Rules

- NEVER ship from a protected branch (`main`/`master`/`develop`) — Step 1 MUST stop
- NEVER force push (`--force` or `--force-with-lease`) — always plain `git push`
- NEVER skip self-review — Step 4 is mandatory every time, no exceptions
- NEVER post review findings to GitHub — findings stay in the conversation only
- NEVER create tasks directly from review findings — route through `/orc:plan` or `/orc:execute <description>`
- Uncommitted changes are advisory only — they may belong to another branch. Do NOT fail on them.
- Incomplete tasks are advisory only — do NOT fail on them
