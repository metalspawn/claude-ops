# Workshop: Agentic Code Workflows with Claude Code

## Overview

**Duration:** 100 minutes
**Format:** Walkthrough + hands-on (everyone drives)
**Prerequisite:** Claude Code installed and authenticated, `orc` plugin installed

This workshop covers the agentic workflow for non-trivial code changes: **collaborative planning** followed by **autonomous execution** with parallel review gates, then **submitting and handling feedback**. By the end, you'll have run through the full cycle yourself.

---

## Workshop Structure

| Time | Section | Mode |
|------|---------|------|
| 0–15 min | **Part 1: Why This Flow Exists** | Presentation |
| 15–35 min | **Part 2: The Components** | Walkthrough |
| 35–50 min | **Part 3: The Detail That Makes It Work** | Deep dive |
| 50–100 min | **Part 4: Hands-On** | Everyone drives |

---

## Part 1: Why This Flow Exists (15 min)

### The Problem

When you ask an AI assistant to "add feature X", it jumps straight to implementation. The result:

- Changes that don't match the project's conventions
- Files in the wrong places
- Naming that makes sense to the implementer but confuses the reader
- No verification that it actually works
- Commits that need to be reverted and redone

This is the same problem we'd have with a new hire who skips the team's PR process. The fix isn't better prompts — it's better process.

### The Solution: Plan → Execute → Submit → Feedback

For larger features that span multiple PRs, `/orc:decompose` breaks them into ordered, independently shippable steps first — then each step runs through the cycle below.

```
/orc:plan        /orc:tasks       /orc:execute        /orc:submit        /orc:pull-comments
┌────────────┐   ┌────────────┐   ┌────────────────┐   ┌────────────┐   ┌─────────────────┐
│            │   │            │   │                │   │            │   │                 │
│ Explore    │OK │ Create     │OK │ For each task: │OK │ Push → PR  │   │ Fetch external  │
│   ──► Plan │──►│ tasks with │──►│  worker        │──►│ Self-review│   │ PR comments     │
│     │      │   │ acceptance │   │    │           │   │ Triage     │   │ Categorise      │
│     ▼      │   │ criteria   │   │    ▼           │   │ findings   │   │ Route           │
│ Plan Review│   │            │   │  Review gates  │   │            │   │                 │
│     │      │   │ Present    │   │  (parallel)    │   │            │   │                 │
│     ▼      │   │ for review │   │    │           │   │            │   │                 │
│ Present to │   │            │   │    ▼           │   │            │   │                 │
│ human      │   │            │   │  All PASS?     │   │            │   │                 │
│            │   │            │   │  → Commit      │   │            │   │                 │
└────────────┘   └────────────┘   └────────────────┘   └─────┬──────┘   └────────┬────────┘
                                                             │                   │
                                                             │   Findings or     │
                                                             │   feedback?       │
                                                             │       │           │
                                                             ▼       ▼           │
                                                        /orc:execute ◄───────────┘
                                                        (fix and re-submit)
```

Three checkpoints give you control:
1. **After plan** — review, adjust, or memorise the plan
2. **After tasks** — review task breakdown and acceptance criteria
3. **Execute** — autonomous from here

Then submit opens the PR, self-reviews it, and pull-comments brings external feedback back into the cycle.

### Why Not Just "Claude, do the thing"?

1. **Plans catch misunderstandings early.** A 2-minute plan review is cheaper than a 20-minute revert.
2. **Tasks create accountability.** Each has acceptance criteria. The validator checks them.
3. **Parallel review catches different classes of problems.** Convention violations, misleading names, and broken tests are orthogonal concerns.

---

## Part 2: The Components (20 min)

### 2.1 The Plugin (`orc`)

The `orc` plugin provides seven skills and five agents:

**Skills (workflow orchestration):**

| Skill | Purpose |
|-------|---------|
| `/orc:decompose` | Break feature into PR-sized steps (for multi-PR work) |
| `/orc:plan` | Explore → Plan → Review → present for approval |
| `/orc:branch` | Create feature branch (auto-invoked by `/orc:tasks` or standalone) |
| `/orc:tasks` | Branch setup → create tasks with acceptance criteria |
| `/orc:execute` | Worker → parallel review gates → commit |
| `/orc:submit` | Push → PR → self-review → triage findings |
| `/orc:pull-comments` | Fetch external PR comments → categorise → triage |

**Agents (specialised roles):**

| Agent | Role | Verdict |
|-------|------|---------|
| `worker` | Implements code | Structured change report |
| `plan-reviewer` | Critiques plans before execution | APPROVED / NEEDS REVISION / REJECTED |
| `code-reviewer` | Checks conventions against project CLAUDE.md | PASS / FAIL |
| `semantic-reviewer` | Checks naming and comment clarity | PASS / PASS WITH ADVISORIES / FAIL |
| `validator` | Runs tests and type checks | PASS / FAIL |

### 2.2 The Global CLAUDE.md

The orchestration layer. It tells Claude *how to work*, not what to build.

Key instruction:
> You are the orchestrator. You coordinate and delegate — you do NOT implement directly.

Without this, Claude will try to do everything itself. The CLAUDE.md also contains a decision table for when to plan vs act directly.

### 2.3 The Project CLAUDE.md

This is what makes the code-reviewer useful. Without it, the reviewer can only check generic framework idioms. With it, the reviewer checks *your* conventions.

Example sections:

```markdown
## File Structure
- Pages live in src/app/[feature]/page.tsx
- Components in src/components/[feature]/ComponentName.tsx
- Server Actions in src/lib/actions/[feature].ts

## Data Fetching
- Use Server Actions for mutations.
- Use Apollo Client for GraphQL queries in Client Components.

## Testing
- Integration tests in __tests__/integration/
- Run with: npm run test:integration
```

The code-reviewer reads this file and checks every changed file against it.

---

## Part 3: The Detail That Makes It Work (15 min)

### 3.1 Acceptance Criteria

The bridge between plan and execution. They serve three agents:
- The **worker** uses them to know when to stop
- The **code-reviewer** and **validator** use them to know what to check
- The **semantic-reviewer** uses the context to judge whether names match intent

**Bad:**
> - Component works correctly
> - Tests pass

**Good:**
> - `UserProfile.tsx` is a Client Component that accepts a `userId: string` prop
> - `UserProfile.tsx` renders the user's name, email, and avatar from the GraphQL query
> - `fetchProfile` server action returns `{ name: string, email: string, avatarUrl: string }`
> - `npm run test:integration` passes with no new failures

### 3.2 The Parallel Review Gates

After the worker finishes, three agents run simultaneously:

| Agent | Checks | Source of Truth |
|-------|--------|-----------------|
| code-reviewer | Structure, conventions, file placement | Project CLAUDE.md |
| semantic-reviewer | Naming clarity, comment accuracy | The code itself |
| validator | Tests, types, acceptance criteria | Project tooling |

**Why parallel?** They check orthogonal concerns. Running sequentially triples review time for no benefit.

**Why all three re-run on failure?** A fix to satisfy the code-reviewer might introduce a misleading name or break a test. Each cycle must verify the complete state.

### 3.3 The Feedback Loop

When a gate fails:
1. Failure report goes back to the orchestrator
2. Orchestrator sends the specific failure to the worker
3. Worker fixes
4. **All three** gates re-run (not just the one that failed)
5. Repeat until all pass

---

## Part 4: Hands-On Exercise (50 min)

### Setup (5 min)

Install the `orc` plugin and verify:

```bash
# Install the plugin
claude plugin install orc

# Verify
claude --print-skills | grep orc
```

Clone the workshop sandbox project:

```bash
# TODO: Replace with actual repo URL
git clone <workshop-sandbox-repo>
cd workshop-sandbox
```

### The Exercise

Work through the full flow on a prepared feature request.

#### Step 1: Plan (~10 min)

```
/orc:plan <feature request>
```

**Watch for:**
- Does the Explore agent find the right files?
- Does the Plan agent reference specific file paths?
- Does the plan-reviewer catch anything?

Approve when satisfied.

#### Step 2: Create Tasks (~5 min)

```
/orc:tasks
```

**Watch for:**
- Does each task have acceptance criteria?
- Are the criteria specific and verifiable?
- Are dependencies set up correctly?

#### Step 3: Execute (~20 min)

```
/orc:execute
```

**Watch for:**
- Does the code-reviewer reference conventions from the project CLAUDE.md?
- Does the semantic-reviewer catch any naming issues?
- Does the validator distinguish new failures from pre-existing ones?
- When a gate fails, does the fix cycle work?

#### Step 4: Submit (~5 min)

```
/orc:submit
```

**Watch for:**
- Does the preflight catch any issues (uncommitted changes, incomplete tasks)?
- Is the PR title derived correctly from the branch name?
- Does the self-review (via `/review`) find anything?
- How are findings categorised (Must Address / Should Consider / Nitpick)?
- Does the triage flow feel natural?

If findings are selected, address them via `/orc:execute <description>` and re-submit.

#### Step 5: Handle PR Feedback (~5 min)

*This step simulates receiving external PR comments. Have a partner (or the facilitator) leave a review comment on your PR, then:*

```
/orc:pull-comments
```

**Watch for:**
- Are comments correctly categorised (Blocking / Suggestion / Question / Nitpick)?
- Does CHANGES_REQUESTED elevate all inline comments to Blocking?
- Are bot comments filtered out?
- Does the routing make sense (questions → reply on PR, code changes → `/orc:plan` or `/orc:execute`)?

#### Step 6: Debrief (~5 min)

After the cycle completes, check:
- The git log — is the commit message conventional?
- The code — does it match the project conventions?
- The test results — do they pass?

### Discussion Points

1. **Where did the flow add value?** What would you have missed without it?
2. **Where did it add friction?** Was any gate unnecessary for this change?
3. **What would you change?** What conventions matter most for your projects?
4. **Project CLAUDE.md:** What conventions should you codify?
5. **Submit workflow:** Did the self-review catch things you'd have missed? Was the PR body useful?

---

## Key Lesson

Descriptive instructions get ignored. "Use the validator agent" is a suggestion. "You MUST spawn the validator. NEVER run tests yourself" is a rule. The agents are written prescriptively with MUST/NEVER throughout because that's what produces consistent behaviour.
