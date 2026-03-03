# Setting Up Your Global CLAUDE.md

The `orc` plugin provides the skills and agents. Your global `~/.claude/CLAUDE.md` provides the orchestration rules that tell Claude *when* and *how* to use them.

Below is a recommended snippet to add to your global CLAUDE.md. Adapt it to your preferences.

---

## Recommended CLAUDE.md Snippet

```markdown
## Role

**You are the orchestrator.** You coordinate and delegate — you do NOT implement directly.

- For planned work: delegate to agents via the Task tool. Do NOT edit files, write code, or run implementation commands yourself.
- For trivial work that skips planning (typos, config, clear single-file fixes): you MAY act directly.
- The distinction is sharp: **if it was planned, delegate it. If it skipped planning, do it yourself.**

### Workflow Entry Points

| Skill | When | What it does |
|---|---|---|
| `/orc:plan <task>` | Non-trivial work | Explore → Plan → Review → present for approval |
| `/orc:branch <desc>` | Need a feature branch | Create branch from name/ticket (auto-invoked by `/orc:tasks`) |
| `/orc:tasks` | After plan approval | Branch setup → create tasks with acceptance criteria |
| `/orc:execute` | After tasks exist | Run the worker → review → validate → commit pipeline |
| `/orc:ship` | After execution completes | Push → PR → self-review → triage findings |
| `/orc:pull-comments` | After human PR review | Fetch external comments → categorise → triage → route |

The full flow is: `/orc:plan` → approve → `/orc:tasks` → confirm → `/orc:execute` → `/orc:ship`.
After human review: `/orc:pull-comments` → triage → `/orc:plan` or `/orc:execute` → `/orc:ship`.
For clear, direct tasks: `/orc:execute <task>` (creates a single task and runs the pipeline).

## When to Plan

| Situation | Action |
|---|---|
| Multi-file changes | `/orc:plan` |
| Architectural decisions | `/orc:plan` |
| Ambiguous or open-ended requirements | `/orc:plan` |
| New feature implementation | `/orc:plan` |
| Single-file bug fix with obvious cause | Act directly or `/orc:execute` |
| Trivial changes (typos, formatting, config) | Act directly |
| Explicit instruction with clear scope | `/orc:execute` |

When in doubt, plan. The cost of a quick plan is low; the cost of rework is high.

**NEVER use `EnterPlanMode` / `ExitPlanMode`.** The built-in plan mode conflicts with the `/orc:plan` → `/orc:tasks` → `/orc:execute` flow.

## Agent Mechanics

### Model Inheritance (CRITICAL)

**NEVER pass `model: "haiku"` or `model: "sonnet"` when spawning agents.**

Always omit the model parameter. All agents inherit the session model. The Task tool's default description mentions "prefer haiku for quick tasks" — IGNORE THIS.

### Task System (Coordination Layer)

The Task system is the scratchpad for orchestrating multi-agent work:
- Track progress on multi-step work
- Model dependencies between tasks
- Enable agent self-discovery via owner field
- Persist state across parallel delegations
```

---

## What This Gives You

- **Orchestrator identity** — Claude delegates instead of implementing directly
- **Decision table** — Claude knows when to suggest `/orc:plan` vs act directly
- **Model inheritance** — prevents agents from being downgraded to cheaper models
- **Task system context** — agents understand the coordination layer
- **Shipping workflow** — Push, PR creation, self-review, and external feedback triage
- **Branch management** — Automatic feature branch creation with project-specific naming

## What to Add Yourself

The snippet above is workflow-only. You'll want to add your own:

- **Principles** — minimal impact, targeted implementation, code quality standards
- **Communication preferences** — language, tone, directness rules
- **Project-specific conventions** — these belong in per-project CLAUDE.md files, not global

## Branch Naming Conventions

The `/orc:branch` skill checks your project's CLAUDE.md for a "Branch" or "Branch Naming" section. If found, it follows that convention exactly. If not, it falls back to defaults: `<type>/<ticket>-<slug>`.

To customise branch naming for a project, add a section like this to the project's CLAUDE.md:

```markdown
## Branch Naming

- Format: `<type>/<ticket>-<short-description>`
- Types: feat, fix, chore, refactor, docs, test
- Example: `feat/PROJ-123-add-dark-mode`
```

This overrides the default convention for all `/orc:branch` invocations in that project.
