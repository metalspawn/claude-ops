---
name: pull-comments
description: "Fetch and triage external PR review comments. Categorises feedback as blocking, suggestion, question, or nitpick, then routes selected items to the appropriate next step."
---

# /orc:pull-comments — Fetch and Triage PR Feedback

## Input

$ARGUMENTS

## Process

Follow these steps in exact order. You run all steps directly — no agent delegation.

### Step 1: Identify PR

If `$ARGUMENTS` contains a PR number or URL, use that.

Otherwise, detect from the current branch:

```
gh pr view --json number,url
```

If no PR is found → STOP and tell the user no PR exists for the current branch.

### Step 2: Fetch comments

Determine the repo owner/name:

```
gh repo view --json nameWithOwner --jq '.nameWithOwner'
```

Get the current user's login:

```
gh api user --jq '.login'
```

Fetch from three GitHub API endpoints using `gh api`:

1. **Standalone review comments:** `repos/{owner}/{repo}/pulls/{number}/comments` — comments on the diff that are not part of a formal review submission.
2. **Review submissions:** `repos/{owner}/{repo}/pulls/{number}/reviews` — capture each review's `state` (APPROVED, CHANGES_REQUESTED, COMMENTED). For each review, also fetch `repos/{owner}/{repo}/pulls/{number}/reviews/{review_id}/comments` for the review's bundled inline comments.
3. **Discussion comments:** `repos/{owner}/{repo}/issues/{number}/comments`

**Filtering — MUST apply before any categorisation:**
- Remove comments from the current user (matched by login)
- Remove comments from users whose login contains "claude" or "bot" (case-insensitive)

If no external comments remain after filtering → STOP and report "No external comments found."

### Step 3: Categorise

**Thread grouping:** Comments with `in_reply_to_id` MUST be grouped under their parent comment, not listed as separate items.

**Reviewer-level elevation:** If ANY review from a reviewer has `CHANGES_REQUESTED` state, ALL inline comments from that reviewer are elevated to **Blocking** regardless of their content.

For all other comments, apply these rules in priority order (first match wins):

| Category | Match condition |
|---|---|
| **Blocking** | Review state is `CHANGES_REQUESTED`, OR body contains "must" / "required" / "blocking" / "please fix" (case-insensitive, word boundaries) |
| **Suggestion** | Body contains "suggest" / "consider" / "could" / "might" / "what about" (case-insensitive) |
| **Question** | Body contains "?" OR starts with "why" / "how" / "what" / "can you explain" (case-insensitive) |
| **Nitpick** | Body contains "nit" / "minor" / "optional" / "style" (case-insensitive) |
| **Default** | If none of the above match → **Suggestion** |

### Step 4: Present and triage

Present a numbered list grouped by category (Blocking first, then Suggestion, Question, Nitpick).

Format each item as:

```
[@reviewer] [file:line] summary
```

Omit `[file:line]` for discussion comments that have no file association.

Use `AskUserQuestion` to let the user select which items to address. Accepted inputs:
- Comma-separated numbers (e.g., "1, 3, 5")
- `"all"` — select every item
- `"blocking"` — select all Blocking items
- `"none"` — select nothing

### Step 5: Next steps

Based on the user's selection:

- **"none" selected:** Done. No further action needed.
- **Questions selected:** Advise the user to reply on the PR directly — these are conversational, not code changes. Do NOT route questions to planning or execution.
- **Code-change items selected (non-trivial — multi-file or architectural):** Summarise the selected items as a brief. Tell the user: "Run `/orc:plan` to plan the changes, then `/orc:tasks` → `/orc:execute` → `/orc:ship`."
- **Code-change items selected (trivial — single-file, obvious fix):** Summarise. Tell the user: "Run `/orc:execute <description>` to address directly, then `/orc:ship`."

If the selection contains a mix (e.g., questions and code changes), present each group's routing separately.

## Rules

- NEVER create tasks directly from PR comments — route through `/orc:plan` or `/orc:execute <description>`
- NEVER auto-respond to PR comments — the user decides what to say
- NEVER modify the PR — no commenting, no resolving conversations, no changing PR state
- NEVER include comments from the current user or bot accounts in the triage list
- NEVER skip the filtering step — bot and self-authored comments pollute triage
- NEVER list thread replies as separate items — group them under the parent comment
