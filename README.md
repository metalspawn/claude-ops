# orc — Orchestrated Agentic Workflows for Claude Code

A Claude Code plugin that provides structured, multi-agent workflows for non-trivial code changes. Plan collaboratively, then execute autonomously with parallel quality gates.

## The Flow

```
/orc:plan → approve → /orc:tasks → confirm → /orc:execute → /orc:ship
                          │                                      │
                   creates branch                     push → PR → self-review → triage
                   (if needed)                               │
                                                    findings? → /orc:plan or /orc:execute → /orc:ship
                                                    clean? → ready for human review
                                                             │
                                                    /orc:pull-comments → triage → /orc:plan or /orc:execute → /orc:ship
```

Seven skills. Five specialised agents.

| Phase | Skill | What happens |
|-------|-------|-------------|
| **Decompose** | `/orc:decompose` | Break a feature into PR-sized steps — clarifies boundaries with you first |
| **Plan** | `/orc:plan` | Explore codebase → clarify ambiguity → produce plan → critical review → present for approval |
| **Branch** | `/orc:branch` | Create a feature branch (invoked by `/orc:tasks` or directly) |
| **Tasks** | `/orc:tasks` | Branch setup → create tasks with acceptance criteria from the plan |
| **Execute** | `/orc:execute` | For each task: worker implements → three parallel review gates → commit |
| **Ship** | `/orc:ship` | Push → create/update PR → self-review → triage findings |
| **Review** | `/orc:pull-comments` | Fetch external PR comments → categorise → triage → route to next step |

For simple, direct tasks: `/orc:execute <task>` skips planning and creates a single task inline.

### For Larger Features

When work spans multiple PRs, start with `/orc:decompose` to break the feature into ordered, independently shippable steps. After approving the decomposition, run each step through the standard `/orc:plan` → `/orc:tasks` → `/orc:execute` → `/orc:ship` cycle.

## Agents

| Agent | Role | Used by |
|-------|------|---------|
| `worker` | Implements code for a specific task | `/orc:execute` |
| `plan-reviewer` | Critiques plans for flaws before execution | `/orc:plan` |
| `code-reviewer` | Checks code against project CLAUDE.md conventions | `/orc:execute` |
| `semantic-reviewer` | Checks naming clarity and comment accuracy | `/orc:execute` |
| `validator` | Runs tests, lint, type checks | `/orc:execute` |

## The Review Gates

After the worker implements each task, three agents run **in parallel**:

```
worker (implement)
       │
       ▼
┌──────────┬───────────┬───────────┐
│  code-   │ semantic- │ validator │
│ reviewer │ reviewer  │           │
└────┬─────┴─────┬─────┴─────┬─────┘
     │           │           │
     ▼           ▼           ▼
  PASS/FAIL   PASS/ADV/   PASS/FAIL
                FAIL

All PASS → commit
Any FAIL → worker fixes → all three re-run
```

They check orthogonal concerns — conventions, naming, and correctness — so all three must re-run after any fix.

## Verification

Plans include a **Verification Plan** — concrete, product-level scenarios that prove the feature works end-to-end. These aren't restated unit tests; they're things a human (or integration test) can run against the system:

- "User clicks logout → redirected to login page"
- "Run `curl localhost:3000/api/health` → 200 OK"
- "Submit form with empty required field → inline validation error appears"

`/orc:tasks` creates a final verification task (blocked by all implementation tasks) that the validator runs after everything is built. Manual-only scenarios are flagged without blocking.

## Interactive Clarification

Both `/orc:decompose` and `/orc:plan` pause after codebase exploration to check in with you when:

- Requirements are ambiguous (2x+ effort difference between interpretations)
- Multiple viable approaches exist with meaningful trade-offs
- Decomposition boundaries aren't obvious from the code
- Acceptance criteria can't be defined from the information available

When everything is clear, the step is skipped automatically — no unnecessary interruptions.

## Shipping

Once all tasks pass their review gates and are committed, `/orc:ship` handles the PR lifecycle:

1. **Push and PR** — pushes the branch and creates (or updates) a pull request
2. **Self-review** — runs the built-in `/review-pr` skill against the PR to catch issues before a human sees them
3. **Triage** — categorises findings by severity and routes them:
   - Findings that need work feed back into `/orc:plan` (for non-trivial changes) or `/orc:execute` (for straightforward fixes), then `/orc:ship` again
   - A clean self-review means the PR is ready for human review

After a human reviews the PR, `/orc:pull-comments` fetches their feedback, categorises it, and routes it the same way — back through `/orc:plan` or `/orc:execute` and then `/orc:ship` to update the PR.

No new agents are needed for shipping. `/orc:ship` delegates to the built-in `/review-pr` skill for self-review.

## Installation

```bash
# From GitHub
claude plugin install metalspawn/claude-ops --scope user
```

### Optional: CLAUDE.md Setup

The plugin works out of the box — invoke any skill directly. Optionally, add the **orchestrator role** to your global `~/.claude/CLAUDE.md` so Claude auto-delegates to the right skill instead of implementing directly. See [docs/claude-md-guide.md](docs/claude-md-guide.md).

### Project CLAUDE.md

For the code-reviewer to be useful, your project needs a `CLAUDE.md` defining conventions: file structure, framework patterns, data fetching approach, testing setup. Without it, the reviewer can only check generic framework idioms.

## Documentation

- [CLAUDE.md Setup Guide](docs/claude-md-guide.md) — How to configure your global CLAUDE.md
- [Workshop](docs/workshop.md) — 100-minute walkthrough of the full flow

## Design Principles

1. **Explicit invocation** — skills are triggered by the user, not inferred
2. **Three checkpoints** — plan, tasks, execute — each a natural pause point
3. **Parallel review** — orthogonal quality gates run simultaneously
4. **Prescriptive rules** — MUST/NEVER over suggestions (descriptive instructions get ignored)
5. **Scope boundaries** — each agent knows exactly what it checks and what it doesn't

## License

MIT
