---
name: harness-task
description: "Anthropic Harness methodology: autonomous multi-sprint execution with Generator/Evaluator separation, file-based handoff, auto-retry, and multi-instance version management. Use for large implementation tasks requiring quality assurance."
argument-hint: "[spec-file-or-description] [--name instance-name] [--threshold 7] [--max-retries 3] [--start-from N] [--skip-eval] [--sprints N] [--eval-weights '{...}'] [--list] [--resume name] [--archive name]"
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, AskUserQuestion, TaskCreate, TaskUpdate, TaskList
effort: max
---

# Harness-Task: Generator-Evaluator Sprint Execution

<harness-task-command>

You are executing the **Anthropic Harness methodology** — an autonomous multi-sprint system where code generation and quality evaluation are strictly separated. This version supports **multi-instance execution** — multiple harness runs can coexist in the same project without conflict.

## Step 0: Parse Arguments

Arguments: $ARGUMENTS

Parse the following from arguments:
- **spec**: A file path to a spec/PRD document, OR a task description string
- **--name <name>**: Instance name (kebab-case, e.g. `auth-refactor`). If omitted, auto-generated from spec filename or first 3 words of description
- **--threshold N**: Evaluation pass threshold (default: 7, range: 1-10)
- **--max-retries N**: Max fix attempts per sprint (default: 3). Total attempts = 1 (initial) + max-retries. Example: `--max-retries 3` means up to 4 total attempts (1 initial + 3 retries).
- **--start-from N**: Start from sprint N (skip prior sprints)
- **--skip-eval**: Skip evaluation (debug mode)
- **--sprints N**: Override number of sprints (default: auto from planning)
- **--eval-weights**: Custom evaluation weights as JSON, e.g. `--eval-weights '{"functionality":40,"reliability":20,"compatibility":20,"quality":20}'`
- **--list**: List all harness instances and their status, then stop (no execution)
- **--resume <name>**: Resume a specific instance from where it left off
- **--archive <name>**: Move a completed instance to `_archived/`, then stop

If no spec file is given and only a description is provided, you will act as Planner first (Phase 0).

### Instance Name Auto-Generation

When `--name` is not provided:
1. If spec is a file path: use the filename without extension, converted to kebab-case (e.g. `auth_migration_plan.md` → `auth-migration-plan`)
2. If spec is a description: take the first 3-5 significant words, kebab-case (e.g. "Refactor the authentication system" → `refactor-auth-system`)
3. If the auto-generated name already exists in the registry, append a numeric suffix: `auth-refactor-2`, `auth-refactor-3`, etc.

### Argument Validation Rules

Before proceeding, validate every parsed argument:

| Argument | Type | Constraint | On Violation |
|---|---|---|---|
| `spec` | string (file path or description) | If a file path: file must exist and be readable | **STOP** with error: "Spec file not found or unreadable: {path}" |
| `--name` | string | Must be kebab-case (lowercase letters, digits, hyphens only), 1-50 chars | **STOP** with error: "Instance name must be kebab-case (a-z, 0-9, hyphens), 1-50 chars, got: {value}" |
| `--threshold` | integer | Must be between 1 and 10 inclusive | **STOP** with error: "Threshold must be an integer between 1 and 10, got: {value}" |
| `--max-retries` | integer | Must be >= 0 | **STOP** with error: "Max retries must be a non-negative integer, got: {value}" |
| `--start-from` | integer | Must be >= 1 | **STOP** with error: "Start-from must be a positive integer, got: {value}" |
| `--sprints` | integer | Must be between 1 and 10 inclusive | **STOP** with error: "Sprints must be an integer between 1 and 10, got: {value}" |
| `--skip-eval` | boolean flag | No value required | N/A |
| `--eval-weights` | JSON object | Must contain exactly 4 keys: `functionality`, `reliability`, `compatibility`, `quality`. All values must be non-negative integers summing to exactly 100. | **STOP** with error describing which constraint failed |
| `--resume` | string | Instance must exist in registry | **STOP** with error: "Instance not found: {name}. Use --list to see available instances." |
| `--archive` | string | Instance must exist and have status `completed` or `aborted` | **STOP** with error describing the issue |

**Mutual exclusions**:
- `--skip-eval` and `--eval-weights` MUST NOT be used together. If both are present, **STOP** with error: "Cannot use --skip-eval and --eval-weights together: --skip-eval disables evaluation, but --eval-weights configures it."
- `--list`, `--resume`, and `--archive` are management commands — they MUST NOT be combined with `spec` or with each other. If mixed, **STOP** with error explaining which flags conflict.

### Spec File Validation

If `spec` is a file path, validate the spec file before any workspace initialization:

1. **Existence check**: Verify the file exists at the given path. If not, **STOP** with error: "Spec file does not exist: {path}".
2. **Readability check**: Attempt to read the file. If it fails (permissions, encoding), **STOP** with error: "Spec file is not readable: {path} — {reason}".
3. **Minimum content check**: The file must contain at least 50 non-whitespace characters. If it does not, **STOP** with error: "Spec file appears empty or too short (< 50 non-whitespace chars): {path}".

Only proceed to Step 0.5 after all argument and spec validations pass.

## Project Root

Determine `{project_root}` using this priority:

1. **Spec git root**: If `spec` is a file path, use the git repository root containing that file (run `git -C <spec_dir> rev-parse --show-toplevel`).
2. **CWD git root**: If the spec is not a file or step 1 failed, use the git repository root of the current working directory (run `git rev-parse --show-toplevel`).
3. **CWD**: If neither git root can be determined, fall back to the current working directory.

All `.harness/` paths are relative to `{project_root}`.

## Step 0.5: Management Commands & Backward Compatibility

### Backward Compatibility Migration

Before any operation, check if a legacy flat `.harness/` structure exists:

1. If `{project_root}/.harness/config.json` exists **AND** there is no `{project_root}/.harness/registry.json`:
   - This is a legacy single-instance layout
   - Create directory `{project_root}/.harness/default/`
   - Move all files/directories (`spec.md`, `config.json`, `summary.md`, `contracts/`, `completions/`, `evaluations/`, `logs/`) into `.harness/default/`
   - Do NOT move `temp/` — it stays at `.harness/temp/`
   - Create `.harness/registry.json` with one entry for the migrated `default` instance (read status from the migrated config.json)
   - Inform the user: "Migrated legacy .harness/ to .harness/default/. Created registry."

### --list: List Instances

If `--list` is specified:
1. Read `{project_root}/.harness/registry.json`
2. Display a table of all instances:
   ```
   Harness Instances:
   | Name | Status | Sprints | Created | Last Updated |
   |------|--------|---------|---------|--------------|
   | auth-refactor | completed | 4/4 passed | 2026-03-30 | 2026-03-30 |
   | ota-integration | running | 2/5 passed | 2026-03-31 | 2026-03-31 |
   ```
3. If no registry exists, display: "No harness instances found."
4. **STOP** — do not proceed to sprint execution.

### --resume: Resume Instance

If `--resume <name>` is specified:
1. Read `{project_root}/.harness/registry.json`, find the instance
2. Read `{project_root}/.harness/{name}/config.json`
3. Determine the resume point:
   - Find the first sprint with status `pending` or `in_progress` or `failed` in the sprints array
   - If all sprints are `passed` or `auto_skip`, report "Instance already completed" and **STOP**
4. Set `startFrom` to that sprint number
5. Load all existing config values (threshold, maxRetries, evalWeights, etc.)
6. Continue to Step 3 (Sprint Execution Loop) using the instance's working directory

### --archive: Archive Instance

If `--archive <name>` is specified:
1. Verify the instance status is `completed` or `aborted`
2. Create `{project_root}/.harness/_archived/` if it doesn't exist
3. Move `{project_root}/.harness/{name}/` to `{project_root}/.harness/_archived/{name}/`
4. Update registry.json: set instance status to `archived`
5. Inform the user and **STOP**

## Step 1: Initialize Workspace

Resolve the instance name `{name}` (from `--name`, `--resume`, or auto-generated).

Create the harness working directory structure:

```
{project_root}/.harness/
├── registry.json                  # Instance registry (create if not exists)
└── {name}/                        # Instance working directory
    ├── spec.md                    # Full specification (input or generated)
    ├── config.json                # Harness configuration
    ├── contracts/                 # Sprint contracts
    │   └── s{N}_contract.md
    ├── completions/               # Generator outputs
    │   └── s{N}_completion.md
    ├── evaluations/               # Evaluator outputs
    │   └── s{N}_evaluation.md
    └── logs/
        └── harness.log            # Execution log
```

**Registry**: Create or update `{project_root}/.harness/registry.json`:
```json
{
  "version": 2,
  "instances": [
    {
      "name": "<instance name>",
      "description": "<first line of spec or description>",
      "status": "initialized",
      "createdAt": "<timestamp>",
      "updatedAt": "<timestamp>",
      "sprintsPassed": 0,
      "sprintsTotal": 0
    }
  ]
}
```

**Instance config**: Write `{project_root}/.harness/{name}/config.json`:
```json
{
  "name": "<instance name>",
  "threshold": <parsed or 7>,
  "maxRetries": <parsed or 3>,
  "startFrom": <parsed or 1>,
  "skipEval": <parsed or false>,
  "evalWeights": {
    "functionality": 30,
    "reliability": 30,
    "compatibility": 20,
    "quality": 20
  },
  "createdAt": "<timestamp>",
  "status": "initialized",
  "sprints": []
}
```

**Global `status` values** (updated as execution progresses):

| Status | Meaning |
|--------|---------|
| `initialized` | Workspace created, not yet started |
| `planning` | Phase 0 planning in progress |
| `running` | Sprint execution loop is active |
| `completed` | All sprints finished (passed or failed) |
| `aborted` | Execution stopped due to unrecoverable error |
| `archived` | Instance moved to _archived/ (registry only) |

**`sprints` array schema** (one entry per sprint, appended during planning, updated during execution):
```json
{
  "sprint": 1,
  "name": "<sprint name>",
  "status": "pending",
  "score": null,
  "retries": 0,
  "duration": null
}
```

Per-sprint `status` values: `pending`, `in_progress`, `passed`, `failed`, `auto_skip`.

**Registry sync**: After every status change (planning → running → completed, sprint pass/fail), update both the instance `config.json` AND the registry.json entry (status, updatedAt, sprintsPassed, sprintsTotal).

## Step 2: Phase 0 — Planning

If a spec file was provided, copy it to `.harness/{name}/spec.md`.

If only a description was given, act as **Planner**:
1. Read the project codebase to understand current architecture
2. Generate a complete technical specification including:
   - Current state analysis
   - Target architecture
   - Data/API inventory
   - Feature degradation matrix (if applicable)
   - Implementation priorities
3. Write the spec to `.harness/{name}/spec.md`

Then **decompose into Sprints**:
1. Analyze the spec and break it into 3-7 sequential sprints
2. Each sprint should be:
   - Self-contained with clear boundaries
   - Buildable on prior sprints
   - Completable in one Claude session (~10min of coding)
3. For each sprint, write a contract file `.harness/{name}/contracts/s{N}_contract.md` containing:
   - Sprint goal (1 sentence)
   - Detailed task description
   - Numbered acceptance criteria (5-12 items, specific and testable)
   - Input dependencies (which prior sprints)

Present the sprint plan to the user for confirmation before proceeding.

## Step 3: Sprint Execution Loop

For each sprint N (from `startFrom` to total):

### 3a-0. Contract Negotiation (optional)

Before launching the Generator, review the sprint contract for feasibility:

1. Read `s{N}_contract.md` and verify:
   - The acceptance criteria are specific and testable (not vague like "improve performance")
   - The scope is achievable in a single session
   - All input dependencies (prior sprints) have status PASSED (not FAILED or AUTO-SKIP)
2. If any issue is found:
   - Present the concern to the user with a proposed amendment
   - Wait for user approval or revision
   - Write the amended contract back to `s{N}_contract.md`
3. If no issues are found, proceed directly to 3a.

This phase is skipped when `--start-from` jumps past a sprint or when re-running a Generator on retry (the contract was already negotiated on the first attempt).

### 3a. Generator Phase

Launch a **subagent** (Agent tool, subagent_type: general-purpose) as the Generator:

**Generator system prompt pattern:**
```
You are a senior engineer executing Sprint {N} of a multi-sprint implementation.
Instance: {name}

[Project Spec]: {contents of .harness/{name}/spec.md}
[Prior Sprint Completions]: {contents of all prior s{M}_completion.md}
[Sprint {N} Contract]: {contents of s{N}_contract.md}
{If retry > 0: [Fix Requirements]: {evaluator feedback}}

RESTRICTION: Do NOT read .harness/{name}/evaluations/. The Generator must never see Evaluator reports
from the current or prior sprints except via the fix feedback provided below. This maintains
strict Generator/Evaluator isolation.

File reading scope: The Generator MAY read any project source file, .harness/{name}/spec.md,
.harness/{name}/config.json, .harness/{name}/contracts/*, and .harness/{name}/completions/* (prior sprints only).
The Generator MUST NOT read .harness/{name}/evaluations/* or .harness/{name}/logs/*.
The Generator MUST NOT read or modify other instances under .harness/.

Instructions:
1. Read relevant source files first to understand existing code
2. List files to change/create with summaries
3. Implement changes using Edit/Write tools directly
4. Write completion report to: {.harness/{name}/completions/s{N}_completion.md}

Completion report format:
# Sprint {N} Completion: {name}
## Changed Files
| Path | Operation | Summary |
|------|-----------|---------|
(One row per file. Operation must be one of: CREATE, MODIFY, DELETE.)
## Acceptance Criteria Check
| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
(Status must be one of: PASS, PARTIAL, FAIL. Evidence format: file:line or a brief description.)
## Remaining Issues
(Bullet list of unresolved items, or "None" if all criteria passed.)
```

### 3b. Evaluator Phase (skip if --skip-eval)

Launch a **separate subagent** (Agent tool, subagent_type: general-purpose) as the Evaluator:

**Evaluator system prompt pattern:**
```
You are an independent QA engineer. You did NOT write this code. Evaluate objectively.
Instance: {name}

[Project Spec]: {contents of .harness/{name}/spec.md}
[Sprint {N} Contract]: {contents of s{N}_contract.md}
[Completion Report]: {contents of s{N}_completion.md}

CRITICAL: You MUST read the actual source files to verify changes exist and are correct.
Do NOT trust the completion report blindly.

File reading scope: The Evaluator MAY read any project source file, .harness/{name}/spec.md,
.harness/{name}/config.json, .harness/{name}/contracts/*, .harness/{name}/completions/s{N}_completion.md (current
sprint only), and .harness/{name}/evaluations/* (prior sprints only, to check for regressions).
The Evaluator MUST NOT read .harness/{name}/completions/ for sprints other than the current one.
The Evaluator MUST NOT read or modify other instances under .harness/.

Score on these dimensions (1-10 each):

1. Functionality (weight: {evalWeights.functionality}%)
   - Check each acceptance criterion: PASS / FAIL / PARTIAL with evidence

2. Reliability (weight: {evalWeights.reliability}%)
   - Does it truly work without dependencies it claims to avoid?
   - Are there hidden assumptions or implicit dependencies?
   - Do fallbacks/error handlers actually work?

3. Compatibility (weight: {evalWeights.compatibility}%)
   - Does it break existing functionality?
   - Are boundaries between old and new code clean?

4. Code Quality (weight: {evalWeights.quality}%)
   - API design, naming, error handling
   - No hardcoded values, magic numbers, or duplication

Output format:
## 1. Functionality: X/10
{evidence}
## 2. Reliability: X/10
{evidence}
## 3. Compatibility: X/10
{evidence}
## 4. Code Quality: X/10
{evidence}
## Weighted Score: X.X/10
Compute as: weighted_score = (f*Wf + r*Wr + c*Wc + q*Wq) / 100
where f=Functionality score, r=Reliability score, c=Compatibility score, q=Quality score,
and Wf/Wr/Wc/Wq are the respective percentage weights from evalWeights.
## Pass (>= {threshold}): YES/NO
## Fix Recommendations
{specific file + issue + fix for each failing item}

Write report to: {.harness/{name}/evaluations/s{N}_evaluation.md}
```

### 3c. Score Check & Retry

1. Read the evaluation report
2. Extract weighted score
3. If score >= threshold: **PASS** -- proceed to Sprint N+1
4. If score < threshold AND retries remaining:
   - Extract fix recommendations from the evaluation report
   - Compose a structured Fix Requirements block for the Generator:
     ```
     [Fix Requirements] (retry {attempt}/{maxRetries}):
     Previous score: {score}/10 (threshold: {threshold})
     Issues:
     1. [{FAIL|PARTIAL}] {criterion} -- {file}:{line} -- {description of what is wrong}
     2. ...
     Required fixes:
     1. {specific action to take, referencing file and location}
     2. ...
     ```
   - Re-run Generator with the Fix Requirements block. The Generator MUST overwrite (not append to) the completion report at `.harness/{name}/completions/s{N}_completion.md`.
   - Re-run Evaluator on the updated completion and source files.
5. If score < threshold AND no retries left:
   - Log failure to `.harness/{name}/logs/harness.log` with sprint number, final score, and retry count
   - Apply failure cascading (see 3c-1 below) to auto-skip dependent sprints
   - Continue autonomously to the next non-skipped sprint (do NOT prompt the user)

**Registry sync**: After each sprint completes (pass or fail), update both config.json and registry.json.

#### 3c-1. Failure Cascading (AUTO-SKIP)

When a sprint N fails (exhausts all retries), any subsequent sprint that **depends on sprint N** (declared via `Input dependencies` in its contract) is automatically marked **AUTO-SKIP**:

1. Identify all sprints whose contract lists sprint N (or any already-skipped sprint) as an input dependency.
2. For each such sprint M, write an evaluation file `.harness/{name}/evaluations/s{M}_evaluation.md` containing:
   ```
   ## AUTO-SKIPPED
   Sprint {M} was automatically skipped because its dependency Sprint {N} failed.
   Score: 0/10 | Status: AUTO-SKIP
   ```
3. Record status `AUTO-SKIP` in the progress table (Step 4).
4. Do NOT launch Generator or Evaluator subagents for auto-skipped sprints.
5. Continue to the next non-skipped sprint.

This prevents wasted execution on sprints that cannot succeed due to missing prerequisites.

### 3d. Progress Reporting

After each Generator/Evaluator cycle, report to user. Measure wall-clock duration for each phase and format as `Xm Ys` (e.g., `2m 34s`):
```
[{name}] Sprint {N}/{total}: {sprint_name}
  Generator: done ({duration, e.g. 3m 12s})
  Evaluator: {score}/10 {PASSED|FAILED} ({duration, e.g. 1m 45s})
  Retries: {used}/{max}
  Sprint duration: {total wall-clock for this sprint, e.g. 5m 22s}
  Cumulative: {passed}/{attempted} sprints passed
```

## Step 4: Final Summary

After all sprints complete, generate `.harness/{name}/summary.md`:

```markdown
# Harness Execution Summary

**Instance**: {name}

| Sprint | Name | Score | Retries | Status |
|--------|------|-------|---------|--------|
| 1 | ... | X.X/10 | N | PASSED |

## Configuration
- Instance: {name}
- Threshold: {threshold}/10
- Eval Weights: {weights}
- Total Duration: {time}

## Evaluation Highlights
{Key findings from evaluators across all sprints}

## Recommended Follow-ups
{Aggregate P0/P1 items from all evaluations that passed but had warnings}
```

Update registry.json with final status (`completed`), sprintsPassed, and sprintsTotal.

## Key Principles (from Anthropic's methodology)

1. **Generator and Evaluator are ALWAYS separate agents** — never self-evaluate
2. **Files are the handoff protocol** — no conversational context leakage
3. **Contracts before code** — agree on acceptance criteria first
4. **Specific over subjective** — score on measurable dimensions, not "is this good?"
5. **Iterate on failure** — feed evaluator feedback back to generator
6. **Context isolation** — each agent gets a fresh context via subagent
7. **Language consistency** — all output (contracts, completion reports, evaluations, summaries) must be written in the same language as the spec
8. **Instance isolation** — each harness run operates in its own directory; agents MUST NOT cross instance boundaries

</harness-task-command>
