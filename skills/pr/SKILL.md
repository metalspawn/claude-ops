---
name: pr
description: "Create or update a pull request with project convention detection. Invoked from /orc:submit or directly."
---

# /orc:pr — Pull Request Creation

## Input

$ARGUMENTS

## Definitions

**Protected branches:** `main`, `master`, `develop`. These are the only branches treated as protected by this skill.

## Process

The orchestrator runs this directly via Bash/tool calls — no agent delegation.

### Step 1: Check for existing PR

Run `gh pr view --json url`.

- If PR exists: report the URL and **STOP**.
- If no PR exists: proceed to Step 2.

### Step 2: Detect PR conventions

Run these checks once before creating the PR.

**Base branch:** Determine the base branch — use `main`. If `origin/main` does not exist, use `master`. This base branch is used for both body content generation and PR creation.

#### 2a: PR title convention

Check these sources in priority order. Use the first match found:

1. **Project CLAUDE.md** — Read the project's CLAUDE.md. Look for a section with heading containing "PR" or "Pull Request" (e.g., "PR Title", "Pull Requests", "Pull Request Convention"). If found, follow that convention exactly. Skip remaining title checks.

2. **PR title checker config** — Check for `.github/pr-title-checker-config.json`. If found, read it and extract the title validation pattern (typically at `CHECKS.regexp`). Derive a title that satisfies this regex. For example, a regex like `^(feat|fix|chore|refactor)(\(\w+\))?: ` means:
   - Allowed types from the alternation group
   - Optional scope in parentheses (must match `\w+` — single word, no hyphens)
   - Colon + space separator

3. **Default** — Conventional Commits format: `type(scope): description`
   - **Type:** Derive from the branch name prefix (`feat/`, `fix/`, `chore/`, etc.). If unclear, default to `feat`.
   - **Scope:** A short domain or service label (e.g., `auth`, `api`, `editor`). NOT a slug of the feature. Scope MUST be a single word (`\w+`). If the right scope isn't obvious from the branch name or task context, use `AskUserQuestion` to ask the user.
   - **Description:** Brief summary in imperative mood, derived from branch name and task context.

#### 2b: PR body convention

Check these paths in order. Use the first match found:

1. `.github/PULL_REQUEST_TEMPLATE.md`
2. `.github/pull_request_template.md`
3. `.github/PULL_REQUEST_TEMPLATE/default.md`
4. `PULL_REQUEST_TEMPLATE.md` (repo root)

**If a template is found:** Use it as a guide. Populate relevant sections with content derived from:
- Commit messages (`git log origin/<base>..HEAD --oneline`)
- Task descriptions and acceptance criteria (from TaskList if available)
- Branch name context

Preserve checkboxes and structural elements where they make sense. Omit sections that don't apply rather than leaving them empty or filling with "N/A".

**If no template is found:** Use the default format:
```
## Summary
<Brief description derived from commit messages and task descriptions>

## Changes
<List of key changes from: git log origin/<base>..HEAD --oneline>

## Test Plan
<Derived from task acceptance criteria if available, otherwise "Manual verification">
```

### Step 3: Create PR

Create the PR using `gh pr create --base <base-branch>` with the title and body from Step 2. Use a HEREDOC to pass the body for correct formatting.

Report the PR URL when done.

---

## Edge Cases

- **No commits ahead of base branch** — report "No commits ahead of base branch — nothing to PR" and STOP.
- **`gh` CLI not authenticated** — report the error and STOP. Do NOT attempt to fix authentication.
- **PR template is malformed or empty** — fall back to the default format.
- **Invoked directly (not from submit)** — still works. Submit's preflight (protected branch check, push) is not repeated here; the caller is responsible.
- **Branch not pushed to remote** — report "Branch is not pushed to remote. Run `git push` first or use `/orc:submit`." and STOP. Do NOT push.
- **`gh pr create` fails** — report the error verbatim and STOP. Do NOT retry.

---

## Examples

| Scenario | Convention Source | PR Title | PR Body |
|---|---|---|---|
| Project has `.github/pr-title-checker-config.json` with `feat\|fix\|chore` types | PR title checker config | `feat(auth): add SSO login` | Default format (no template) |
| Project has `.github/PULL_REQUEST_TEMPLATE.md` | PR template file | Default Conventional Commits | Template populated with commit content |
| Project CLAUDE.md has `## Pull Requests` section | Project CLAUDE.md | Follows CLAUDE.md convention | Default format |
| No conventions found | Default | `feat(api): add user search endpoint` | Default Summary/Changes/Test Plan |

---

## Rules

- NEVER create a PR from a protected branch (`main`, `master`, `develop`)
- NEVER ignore a PR template if one exists — use it as a guide
- NEVER force a template section that doesn't apply — omit it instead
- NEVER delegate this to an agent — the orchestrator runs it directly
- ALWAYS check for PR conventions before creating — Step 2 is mandatory
- ALWAYS check for an existing PR first — Step 1 prevents duplicates
