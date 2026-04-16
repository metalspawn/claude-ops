---
name: submit
description: "Push current branch, create or update a PR, run self-review, and triage findings. Run after /orc:execute completes."
---

# /orc:submit — Submit for Review

## Input

$ARGUMENTS

---

## Process

Follow these steps in exact order. Do NOT skip ahead. Steps 1–3 are run directly by the orchestrator via Bash/tool calls. Steps 4–7 handle review, triage, CI, and next steps.

### Step 1: Preflight

Run these checks before doing anything else.

**Branch check:**
- Get current branch: `git branch --show-current`
- STOP immediately if branch is `main`, `master`, or `develop`. Tell the user: "Cannot submit from a protected branch. Switch to a feature branch first."

**Commits ahead check:**
- Run `git log origin/main..HEAD --oneline`. If `origin/main` does not exist, try `origin/master`.
- STOP if no commits are ahead. Tell the user: "No commits ahead of base branch — nothing to submit."

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

- Run the built-in `/review` skill by calling: `Skill(skill: "review")`
- Do NOT spawn a code-reviewer or semantic-reviewer agent. Do NOT use the Agent tool for this step. The ONLY correct action is `Skill(skill: "review")`.
- The review findings are returned directly in the conversation — they are NOT posted to GitHub.
- Parse the returned findings into structured items. Each item MUST have:
  - **file** — the file path
  - **line** — the line number (or range)
  - **description** — what the finding is

### Step 5: Triage findings

If no findings from the review, skip to Step 7 with "no findings".

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

**Present the numbered list, then wait for the caller to specify which findings to address** (e.g., "all", "none", specific numbers). If the caller has already specified a default (e.g., via CLAUDE.md or arguments), apply that default without pausing.

### Step 6: Watch CI

After triage is resolved, check whether CI is running on the branch.

**Find the CI run:**
```bash
gh run list --branch $(git branch --show-current) --limit 1 --json databaseId,status,conclusion,name
```

- If no runs found: skip with "No CI workflows detected for this branch." Proceed to Step 7.
- If the run is already completed with success: report "CI passed" and proceed to Step 7.
- If the run is already completed with failure: go to the failure diagnosis below.
- If the run is in progress or queued: watch it:

```bash
gh run watch <id> --exit-status
```

**On success:** report "CI passed ✓" and proceed to Step 7.

**On failure:** diagnose immediately.

1. Get the failed logs:
```bash
gh run view <id> --log-failed
```

2. Read the error output and diagnose the root cause.
3. Present the diagnosis to the user and recommend next steps:
   - Trivial fix (config, typo, missing env var): "Run `/orc:execute <description>` to fix, then `/orc:submit`."
   - Non-trivial fix (logic error, test failure, dependency issue): "Run `/orc:plan` to plan the fix, then `/orc:tasks` → `/orc:execute` → `/orc:submit`."
4. **You MUST stop here and wait for the user to decide.** Do NOT proceed to Step 7 after a CI failure.

### Step 7: Next steps

This step runs ONLY after triage (Step 5) and CI (Step 6) are resolved. Based on the findings selection:

- **Findings selected (non-trivial — multi-file or architectural):**
  Summarise the selected findings as a brief, then invoke `/orc:plan` to plan the changes. The orchestrator continues the full pipeline from there.

- **Findings selected (trivial — single-file, obvious fix):**
  Summarise the selected findings, then invoke `/orc:execute <description>` to address directly. After execution completes, re-invoke `/orc:submit` to push, re-review, and check CI.

- **No findings or "none" selected:**
  "PR is ready for human review." Report the PR URL.

---

## Rules

- NEVER submit from a protected branch (`main`/`master`/`develop`) — Step 1 MUST stop
- NEVER force push (`--force` or `--force-with-lease`) — always plain `git push`
- NEVER skip self-review — Step 4 is mandatory every time, no exceptions
- NEVER use the Agent tool for self-review — Step 4 MUST use `Skill(skill: "review")`, not a code-reviewer or semantic-reviewer agent
- NEVER post review findings to GitHub — findings stay in the conversation only
- NEVER create tasks directly from review findings — route through `/orc:plan` or `/orc:execute <description>`
- Uncommitted changes are advisory only — they may belong to another branch. Do NOT fail on them.
- Incomplete tasks are advisory only — do NOT fail on them
