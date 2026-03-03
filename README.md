# orc — Orchestrated Agentic Workflows for Claude Code

A Claude Code plugin that provides structured, multi-agent workflows for non-trivial code changes. Plan collaboratively, then execute autonomously with parallel quality gates.

## The Flow

```
/orc:plan <task>  →  approve  →  /orc:tasks  →  confirm  →  /orc:execute
```

Three checkpoints. Three skills. Five specialised agents.

| Phase | Skill | What happens |
|-------|-------|-------------|
| **Plan** | `/orc:plan` | Explore codebase → produce plan → critical review → present for approval |
| **Tasks** | `/orc:tasks` | Create tasks with acceptance criteria from the approved plan |
| **Execute** | `/orc:execute` | For each task: worker implements → three parallel review gates → commit |

For simple, direct tasks: `/orc:execute <task>` skips planning and creates a single task inline.

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

## Installation

```bash
# From GitHub
claude plugin install metalspawn/claude-ops --scope user
```

### CLAUDE.md Setup

The plugin provides the skills and agents. Your global `~/.claude/CLAUDE.md` provides the orchestration rules. See [docs/claude-md-guide.md](docs/claude-md-guide.md) for the recommended setup.

The key addition is the **orchestrator role** — without it, Claude will try to implement directly instead of delegating to agents.

### Project CLAUDE.md

For the code-reviewer to be useful, your project needs a `CLAUDE.md` defining conventions: file structure, framework patterns, data fetching approach, testing setup. Without it, the reviewer can only check generic framework idioms.

## Documentation

- [CLAUDE.md Setup Guide](docs/claude-md-guide.md) — How to configure your global CLAUDE.md
- [Workshop](docs/workshop.md) — 90-minute walkthrough of the full flow

## Design Principles

1. **Explicit invocation** — skills are triggered by the user, not inferred
2. **Three checkpoints** — plan, tasks, execute — each a natural pause point
3. **Parallel review** — orthogonal quality gates run simultaneously
4. **Prescriptive rules** — MUST/NEVER over suggestions (descriptive instructions get ignored)
5. **Scope boundaries** — each agent knows exactly what it checks and what it doesn't

## License

MIT
