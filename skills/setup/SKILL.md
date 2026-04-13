---
name: setup
description: "Audit project infrastructure for orc workflow compatibility and set up missing CLAUDE.md sections."
---

# /orc:setup — Project Setup Audit

## Input

$ARGUMENTS

---

## Process

The orchestrator runs this directly — no agent delegation. Run all checks, present a summary, then offer to fix what's missing.

### Step 1: Preflight

- Run `git rev-parse --git-dir`. If it fails, STOP: "This directory is not a git repository."

### Step 2: Detect commit conventions

Check for commitlint config at the repo root (first match wins):
- `.commitlintrc.json`, `.commitlintrc.yml`, `.commitlintrc.yaml`
- `.commitlintrc.js`, `.commitlintrc.cjs`, `.commitlintrc.mjs`
- `commitlint.config.js`, `commitlint.config.cjs`, `commitlint.config.mjs`, `commitlint.config.ts`

If found, identify the preset:
- `extends: ["@commitlint/config-conventional"]` → "conventional commits"
- `extends: ["@commitlint/config-angular"]` → "angular"
- Other → "custom"

Check for a commit-msg hook (first match wins):
- `.husky/commit-msg`
- `lefthook.yml` or `lefthook.yaml` — look for a `commit-msg` key
- `.git/hooks/commit-msg` — must be executable and not the sample file

Record: convention name (or "none detected"), hook enforced (yes/no).

### Step 3: Detect PR template

Check these paths in order:
1. `.github/pull_request_template.md`
2. `.github/PULL_REQUEST_TEMPLATE.md`
3. `.github/PULL_REQUEST_TEMPLATE/default.md`
4. `docs/pull_request_template.md`
5. `PULL_REQUEST_TEMPLATE.md` (repo root)

If found, read the file and extract section headings (lines starting with `#` or `##`).

Record: path (or "none"), headings list.

### Step 4: Detect pre-commit hooks

Check for a hook manager (first match wins):
- `.husky/pre-commit` → husky
- `lefthook.yml` or `lefthook.yaml` with a `pre-commit` key → lefthook
- `.pre-commit-config.yaml` → pre-commit (Python ecosystem)
- `.git/hooks/pre-commit` (executable, not sample) → native git hook

Record: manager name (or "none"), wired (yes/no).

### Step 5: Local dev readiness

Two layers: can we find the commands, and do they work?

**Detect stack and commands:**

| Files Present | Stack | Test Command | Build/Type Check |
|---|---|---|---|
| `package.json` | Node/JS/TS | `scripts.test` value | `scripts.typecheck` or `scripts.build` value |
| `Gemfile` | Ruby/Rails | `bundle exec rails test` or `bundle exec rspec` | N/A |
| `pyproject.toml` / `setup.cfg` | Python | `pytest` or `python -m pytest` | `mypy .` or `pyright` |
| `go.mod` | Go | `go test ./...` | `go vet ./...` |
| `Cargo.toml` | Rust | `cargo test` | `cargo check` |
| `Makefile` | Generic | `make test` (if target exists) | `make build` (if target exists) |

If `package.json` exists, read `scripts` to find exact commands. For other stacks, use the conventional commands.

**Check dependencies installed:**
- Node: `node_modules/` exists
- Ruby: `vendor/bundle/` exists or `bundle check` succeeds
- Python: `.venv/` or `venv/` exists, or `pip check` succeeds
- Go: `go.sum` exists (modules downloaded on demand)
- Rust: `target/` exists or dependencies download on build

**Check environment:**
- Look for `.env`, `.env.local`, `.env.development`, or similar at repo root
- If none found, check if `.env.example` or `.env.sample` exists (template but not populated)

**Run smoke checks** (only if dependencies are installed):
- Run the detected test command. Record: passes, fails, or errors (e.g., missing DB, missing env var).
- Run the detected build/type check command. Record: passes or fails.
- If either fails, capture the first error line for the report.

**Check visual verification:**
- Check if `mcp__plugin_chrome-devtools-mcp_chrome-devtools__take_screenshot` tool is available.
- If available and a dev server command is detected (e.g., `scripts.dev` in package.json, `bin/rails server`): note that visual verification is possible.

Record: stack, test command + result, build command + result, dependencies installed (yes/no), env file (yes/no/template only), visual verification (available/not available).

### Step 6: Audit CLAUDE.md

Read the project's `CLAUDE.md` at the repo root. If it does not exist, record all three sections as missing.

If it exists, scan headings (case-insensitive) for:
- **Branch section:** heading containing "branch"
- **Commit section:** heading containing "commit"
- **PR section:** heading containing "pr" or "pull request"

Record: file exists (yes/no), which sections are present.

### Step 7: Present summary

Format as a checklist:

```
## Project Setup Report

### Commit Conventions
[✓/✗] Convention: [name or "none detected"]
[✓/✗] Hook enforcement: [manager name or "no commit-msg hook"]

### PR Template
[✓/✗] [path or "no PR template found"]

### Pre-commit Hooks
[✓/✗] [manager + "wired" or "no pre-commit hooks found"]

### Local Dev Readiness
Stack: [detected stack]
[✓/✗] Dependencies installed
[✓/✗] Tests: [command] — [pass/fail/error summary]
[✓/✗] Build/Types: [command] — [pass/fail/error summary]
[✓/✗] Environment: [.env found / .env.example only / none]
[✓/✗] Visual verification: [available / not available]

### CLAUDE.md
[✓/✗] Branch Naming section
[✓/✗] Commit Messages section
[✓/✗] Pull Requests section
```

If everything passes: "All checks passed — project is fully configured for orc workflows." STOP.

### Step 8: Offer remediation

Number each missing item. Use `AskUserQuestion`:
"Would you like to set up the missing items? (all / comma-separated numbers / none)"

If "none": "No changes made." STOP.

### Step 9: Remediate

For each selected item:

**Missing CLAUDE.md sections:**
- If CLAUDE.md does not exist, create it with all missing sections.
- If it exists, append the missing sections.
- Populate from detected conventions (Steps 2-5). If no convention was detected, use defaults.
- Show the user what will be added before writing. Ask for confirmation.

Section templates:

Branch Naming (default):
```
## Branch Naming
- Format: `<type>/<ticket>-<short-description>`
- Types: feat, fix, chore, refactor, docs, test
```

Commit Messages (populated from detection, or default):
```
## Commit Messages
- Format: Conventional Commits — `type(scope): description`
- Types: feat, fix, chore, refactor, docs, test, ci, build, perf
```

Pull Requests (populated from detection, or default):
```
## Pull Requests
- Title: `type(scope): description` matching commit convention
- Scope: single-word domain label (e.g., auth, api, editor)
```

**Local dev issues:**
- Dependencies not installed: print the install command for the detected stack (e.g., `npm install`, `bundle install`, `pip install -e .`)
- Tests failing: report the first error. If it's a missing env var or service, flag it specifically.
- No env file but `.env.example` exists: "Copy `.env.example` to `.env` and populate the values."
- No env file at all: "No environment file found. Check if the project needs one."
- Visual verification not available: "Add Chrome DevTools MCP server to your Claude Code configuration for screenshot-based verification."

**Missing tooling (commit convention, PR template, pre-commit hooks):**
Do NOT create config files. Print specific setup recommendations:

- No commitlint: "Install commitlint: `npm install -D @commitlint/{cli,config-conventional}` and create `.commitlintrc.json` with `{ \"extends\": [\"@commitlint/config-conventional\"] }`"
- No commit-msg hook: "Add a commit-msg hook via your hook manager to run `commitlint --edit $1`"
- No PR template: "Create `.github/pull_request_template.md` with your preferred sections"
- No pre-commit hooks: "Consider husky (Node) or lefthook (polyglot) for pre-commit hook management"

---

## Rules

- NEVER overwrite existing config files or CLAUDE.md content — only create or append
- NEVER install packages — print recommendations only
- NEVER delegate to agents — the orchestrator runs this directly
- NEVER skip the summary — always present the full report before offering remediation
- ALWAYS ask before creating or modifying any file
- ALWAYS use detected conventions to populate CLAUDE.md sections — do not guess
- Idempotent — safe to run multiple times, only reports and acts on what is actually missing
