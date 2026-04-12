# Setting Up Your Global CLAUDE.md

The `orc` plugin works out of the box — invoke any skill directly (`/orc:plan`, `/orc:execute`, etc.) and it handles the rest.

The optional CLAUDE.md setup below adds **auto-delegation**: Claude will reach for the right skill on its own instead of waiting for you to invoke it. Without it, you drive the flow manually.

---

## Optional: Orchestrator Role

Add this to your global `~/.claude/CLAUDE.md` if you want Claude to automatically delegate to `orc` skills.

```markdown
## Role

**You are the orchestrator.** You coordinate and delegate — you do NOT implement directly.

- For planned work: delegate to agents via the Task tool. Do NOT edit files, write code, or run implementation commands yourself.
- For trivial work that skips planning (typos, config, clear single-file fixes): you MAY act directly.
- The distinction is sharp: **if it was planned, delegate it. If it skipped planning, do it yourself.**

### Workflow Entry Points

| Skill | When | What it does |
|---|---|---|
| `/orc:decompose <feature>` | Multi-PR features | Break into PR-sized steps → each step uses the full flow below |
| `/orc:plan <task>` | Non-trivial work | Explore → Plan → Review → present for approval |
| `/orc:branch <desc>` | Need a feature branch | Create branch from name/ticket (auto-invoked by `/orc:tasks`) |
| `/orc:tasks` | After plan approval | Branch setup → create tasks with acceptance criteria |
| `/orc:execute` | After tasks exist | Run the worker → review → validate → commit pipeline |
| `/orc:ship` | After execution completes | Push → PR → self-review → triage findings |
| `/orc:pull-comments` | After human PR review | Fetch external comments → categorise → triage → route |

The full flow is: `/orc:plan` → approve → `/orc:tasks` → confirm → `/orc:execute` → `/orc:ship`.
For multi-PR features: `/orc:decompose` → approve → then for each step: `/orc:plan` → `/orc:tasks` → `/orc:execute` → `/orc:ship`.
After human review: `/orc:pull-comments` → triage → `/orc:plan` or `/orc:execute` → `/orc:ship`.
For clear, direct tasks: `/orc:execute <task>` (creates a single task and runs the pipeline).

## When to Plan

| Situation | Action |
|---|---|
| Multi-PR feature | `/orc:decompose` |
| Multi-file changes | `/orc:plan` |
| Architectural decisions | `/orc:plan` |
| Ambiguous or open-ended requirements | `/orc:plan` |
| Single PR-sized change | `/orc:plan` |
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

## What the Orchestrator Role Gives You

- **Auto-delegation** — Claude chooses the right skill instead of implementing directly
- **Decision table** — Claude knows when to suggest `/orc:plan` vs act directly
- **Model inheritance** — prevents agents from being downgraded to cheaper models
- **Task system context** — agents understand the coordination layer

Without it, the plugin still works — you just invoke skills yourself.

## What to Add Yourself

The snippet above is workflow-only. You'll want to add your own:

- **Principles** — minimal impact, targeted implementation, code quality standards
- **Communication preferences** — language, tone, directness rules
- **Project-specific conventions** — these belong in per-project CLAUDE.md files, not global

## Pre-Commit Hooks for Lint & Format

Linting and formatting are deterministic and mechanical — they should run as pre-commit hooks with auto-fix, not as agent validation. This keeps agent cycles focused on tests and type checks (which require reasoning), and prevents expensive re-runs of all review gates for trivial lint fixes.

Pre-commit hooks also solve an agent-boundary problem: hooks configured in the parent session don't fire for tool calls made by subagents, but pre-commit hooks fire on every `git commit` regardless.

### Recommended Setup

Use a hook manager. Two good options:

| Tool | Config file | Install |
|------|------------|---------|
| [Lefthook](https://github.com/evilmartians/lefthook) | `lefthook.yml` | `brew install lefthook && lefthook install` |
| [Husky](https://typicode.github.io/husky/) | `.husky/pre-commit` | `npx husky init` |

### Example: lefthook.yml

```yaml
pre-commit:
  commands:
    lint-fix:
      glob: "*.{js,jsx,ts,tsx}"
      run: npx eslint --fix {staged_files} && git add {staged_files}
    format:
      glob: "*.{js,jsx,ts,tsx,json,css,md}"
      run: npx prettier --write {staged_files} && git add {staged_files}
```

### Common Auto-Fix Commands

| Language | Lint | Format |
|----------|------|--------|
| JS/TS | `eslint --fix` | `prettier --write` |
| Python | `ruff check --fix` | `ruff format` |
| Ruby | `rubocop -A` | `rubocop -A` (combined) |
| Go | `golangci-lint run --fix` | `gofmt -w` |

### What Happens Without Hooks

Without pre-commit hooks, lint and format issues will **not** be caught by the agent workflow. The validator agent does not run linters — it focuses on tests and type checks. Lint issues will only surface during human PR review or CI checks.

This is a deliberate trade-off: lint enforcement is an infrastructure concern, not an agent concern. If your project doesn't have pre-commit hooks or CI lint checks, add them.

### How Auto-Fix Works

Auto-fix means the hook fixes problems silently on commit. The agent never sees lint failures because they are resolved before the commit lands. If a lint rule cannot auto-fix (e.g., unused variable), it surfaces as a commit failure that the worker must address — but this is rare and fast compared to running a full validation gate.

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
