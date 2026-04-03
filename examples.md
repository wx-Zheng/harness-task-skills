# Harness Skill Usage Examples

## Example 1: With Existing Spec File (Named Instance)

```
/harness-task docs/offline_spec.md --name offline-mvp --threshold 7 --max-retries 3
```

Reads the spec, creates instance `offline-mvp` under `.harness/offline-mvp/`, auto-decomposes into sprints, and executes the full Generator-Evaluator loop.

## Example 2: From Description (Auto-Plan, Auto-Named)

```
/harness-task "Refactor the authentication system from session-based to JWT, maintaining backward compatibility for 2 weeks"
```

Claude acts as Planner first, generates a full spec, then executes sprints. Instance auto-named as `refactor-auth-system`.

## Example 3: Strict Quality Bar

```
/harness-task docs/migration_plan.md --name db-migration --threshold 9 --max-retries 5
```

Higher threshold = more iterations, higher quality. Good for critical systems.

## Example 4: Quick Iteration (Low Bar)

```
/harness-task "Add dark mode support to all components" --threshold 6 --max-retries 1
```

Fast execution with lower quality bar. Good for prototyping. Auto-named as `add-dark-mode`.

## Example 5: Resume from Sprint 3

```
/harness-task docs/spec.md --name auth-refactor --start-from 3
```

Skips Sprint 1-2 (assumes already completed), starts from Sprint 3 within the `auth-refactor` instance.

## Example 6: Custom Evaluation Weights

```
/harness-task docs/spec.md --name mvp-dashboard --eval-weights '{"functionality":50,"reliability":20,"compatibility":20,"quality":10}'
```

Emphasize functionality over code quality (e.g., for MVP/prototype).

### Recommended Weight Configurations

| Scenario | functionality | reliability | compatibility | quality | Rationale |
|---|---|---|---|---|---|
| MVP / Prototype | 50 | 20 | 20 | 10 | Prioritize working features; quality and compatibility are secondary for throwaway code |
| Production Migration | 20 | 30 | 30 | 20 | Compatibility with existing systems and reliability are critical; cannot break production |
| Refactoring | 10 | 20 | 20 | 50 | Primary goal is code quality improvement; functionality should not change |
| Security-Critical | 20 | 40 | 20 | 20 | Reliability (incl. error handling, input validation, edge cases) is paramount |

**Usage examples for each scenario:**

```
# MVP / Prototype
/harness-task "Build a proof-of-concept dashboard" --name poc-dashboard --eval-weights '{"functionality":50,"reliability":20,"compatibility":20,"quality":10}' --threshold 6

# Production Migration
/harness-task docs/db_migration.md --name prod-migration --eval-weights '{"functionality":20,"reliability":30,"compatibility":30,"quality":20}' --threshold 8

# Refactoring
/harness-task docs/refactor_plan.md --name code-cleanup --eval-weights '{"functionality":10,"reliability":20,"compatibility":20,"quality":50}' --threshold 8

# Security-Critical
/harness-task docs/auth_overhaul.md --name auth-security --eval-weights '{"functionality":20,"reliability":40,"compatibility":20,"quality":20}' --threshold 9 --max-retries 5
```

## Example 7: Skip Evaluation (Debug)

```
/harness-task docs/spec.md --name debug-run --skip-eval
```

Only runs Generator, skips Evaluator. Useful for debugging sprint contracts.

## Example 8: List All Instances

```
/harness-task --list
```

Output:
```
Harness Instances:
| Name | Status | Sprints | Created | Last Updated |
|------|--------|---------|---------|--------------|
| auth-refactor | completed | 4/4 passed | 2026-03-28 | 2026-03-28 |
| ota-integration | running | 2/5 passed | 2026-03-30 | 2026-03-31 |
| db-migration | failed | 1/3 passed | 2026-03-31 | 2026-03-31 |
```

## Example 9: Resume a Specific Instance

```
/harness-task --resume ota-integration
```

Reads the instance's config, finds the first pending/failed sprint, and resumes from there. All original settings (threshold, eval-weights, etc.) are preserved.

## Example 10: Archive a Completed Instance

```
/harness-task --archive auth-refactor
```

Moves `.harness/auth-refactor/` to `.harness/_archived/auth-refactor/` and updates the registry. Keeps files accessible but declutters the active instances.

## Example 11: Multiple Instances in Same Project

Run multiple harness tasks on the same codebase without conflicts:

```
# First task: backend refactoring
/harness-task docs/backend_refactor.md --name backend-refactor --threshold 8

# Second task (in a different conversation): frontend overhaul
/harness-task docs/frontend_spec.md --name frontend-overhaul --threshold 7

# Check both:
/harness-task --list
```

Directory structure:
```
.harness/
├── registry.json
├── backend-refactor/
│   ├── spec.md
│   ├── config.json
│   ├── contracts/
│   ├── completions/
│   ├── evaluations/
│   └── logs/
└── frontend-overhaul/
    ├── spec.md
    ├── config.json
    ├── contracts/
    ├── completions/
    ├── evaluations/
    └── logs/
```

## Example 12: Failure Recovery with --resume

When a prior harness run failed partway through (e.g., Sprint 2 failed after exhausting retries), fix the issue manually and resume:

```
/harness-task --resume db-migration
```

This automatically detects Sprint 2 as the first non-passed sprint and resumes from there. Compare with the old `--start-from` approach — `--resume` is smarter because it reads the instance state.

You can still use `--start-from` for explicit control:
```
/harness-task docs/spec.md --name db-migration --start-from 3
```

### AUTO-SKIP Cascading Output

When a sprint fails and later sprints depend on it, the harness automatically skips those dependent sprints:

```
[db-migration] Sprint 1/5: Database Schema Migration
  Generator: done (3m 12s)
  Evaluator: 8.2/10 PASSED (1m 45s)
  Retries: 0/3
  Sprint duration: 5m 22s
  Cumulative: 1/1 sprints passed

[db-migration] Sprint 2/5: API Endpoint Implementation
  Generator: done (4m 08s)
  Evaluator: 5.1/10 FAILED (2m 03s)
  Retries: 1/3
  Generator: done (3m 44s)
  Evaluator: 5.8/10 FAILED (1m 58s)
  Retries: 2/3
  Generator: done (3m 22s)
  Evaluator: 6.0/10 FAILED (1m 50s)
  Retries: 3/3
  Sprint duration: 22m 10s
  Cumulative: 1/2 sprints passed

[db-migration] Sprint 3/5: Frontend Integration
  AUTO-SKIPPED: dependency Sprint 2 failed
  Cumulative: 1/3 sprints passed

[db-migration] Sprint 4/5: Authentication Middleware
  AUTO-SKIPPED: dependency Sprint 2 failed
  Cumulative: 1/4 sprints passed

[db-migration] Sprint 5/5: End-to-End Tests
  Generator: done (2m 55s)
  Evaluator: 7.4/10 PASSED (1m 30s)
  Retries: 0/3
  Sprint duration: 4m 38s
  Cumulative: 2/5 sprints passed
```

After this run, fix Sprint 2 manually and resume with `--resume db-migration`.

## Example 13: Argument Validation Errors

The harness validates all arguments before execution. Here are common validation errors:

### Instance name invalid
```
/harness-task docs/spec.md --name "My Task!"
```
Output:
```
Instance name must be kebab-case (a-z, 0-9, hyphens), 1-50 chars, got: My Task!
```

### Resume non-existent instance
```
/harness-task --resume nonexistent-task
```
Output:
```
Instance not found: nonexistent-task. Use --list to see available instances.
```

### Mixing management commands
```
/harness-task --list --resume auth-refactor
```
Output:
```
Cannot combine management commands: --list and --resume are mutually exclusive and cannot be used with spec.
```

### Threshold out of range

```
/harness-task docs/spec.md --threshold 15
```

Output:
```
Threshold must be an integer between 1 and 10, got: 15
```

### skip-eval and eval-weights conflict

```
/harness-task docs/spec.md --skip-eval --eval-weights '{"functionality":40,"reliability":20,"compatibility":20,"quality":20}'
```

Output:
```
Cannot use --skip-eval and --eval-weights together: --skip-eval disables evaluation, but --eval-weights configures it.
```

## Example 14: Backward Compatibility (Legacy Migration)

If you have an existing flat `.harness/` from a previous version:

```
# Old structure:
.harness/
├── spec.md
├── config.json
├── contracts/
├── completions/
└── evaluations/

# Running any harness command triggers auto-migration:
/harness-task --list

# Output:
Migrated legacy .harness/ to .harness/default/. Created registry.

Harness Instances:
| Name | Status | Sprints | Created | Last Updated |
|------|--------|---------|---------|--------------|
| default | completed | 5/5 passed | 2026-03-25 | 2026-03-28 |

# New structure:
.harness/
├── registry.json
└── default/
    ├── spec.md
    ├── config.json
    ├── contracts/
    ├── completions/
    └── evaluations/
```

## Example 15: Chinese Language Spec

When the spec is written in Chinese, all harness output follows the language consistency principle:

```
/harness-task docs/chinese_spec.md --name auth-jwt-migration --threshold 7
```

### Sample Progress Output

```
[auth-jwt-migration] Sprint 1/3: Database Schema & Token Storage
  Generator: done (2m 48s)
  Evaluator: 8.0/10 PASSED (1m 32s)
  Retries: 0/3
  Sprint duration: 4m 35s
  Cumulative: 1/1 sprints passed
```

## Example 16: End-to-End Walkthrough

Complete harness run with named instance:

### Step 1: Invoke

```
/harness-task "Add a notification system with email and in-app channels" --name notification-system --threshold 7 --max-retries 2
```

### Step 2: Planning

```
Planning: Analyzing codebase and generating specification...
Planning: Spec written to .harness/notification-system/spec.md
Planning: Decomposed into 4 sprints

Sprint Plan:
  Sprint 1: Notification Data Model & Storage
  Sprint 2: Email Channel Integration
  Sprint 3: In-App Channel & User Preferences
  Sprint 4: Batching Engine & Delivery Pipeline

Proceed? (Y/n)
```

### Step 3: Execution

```
[notification-system] Sprint 1/4: Notification Data Model & Storage
  Generator: done (3m 05s)
  Evaluator: 8.1/10 PASSED (1m 40s)
  Sprint duration: 5m 10s
  Cumulative: 1/1 sprints passed

[notification-system] Sprint 2/4: Email Channel Integration
  Generator: done (4m 22s)
  Evaluator: 5.9/10 FAILED (2m 15s)
  Retries: 1/2
  Generator: done (3m 10s)
  Evaluator: 7.3/10 PASSED (1m 50s)
  Sprint duration: 12m 02s
  Cumulative: 2/2 sprints passed

[notification-system] Sprint 3/4: In-App Channel & User Preferences
  Generator: done (3m 48s)
  Evaluator: 7.6/10 PASSED (1m 38s)
  Sprint duration: 5m 41s
  Cumulative: 3/3 sprints passed

[notification-system] Sprint 4/4: Batching Engine & Delivery Pipeline
  Generator: done (4m 55s)
  Evaluator: 7.0/10 PASSED (2m 05s)
  Sprint duration: 7m 25s
  Cumulative: 4/4 sprints passed
```

### Step 4: Final directory

```
.harness/
├── registry.json
└── notification-system/
    ├── spec.md
    ├── config.json
    ├── contracts/
    │   ├── s1_contract.md
    │   ├── s2_contract.md
    │   ├── s3_contract.md
    │   └── s4_contract.md
    ├── completions/
    │   ├── s1_completion.md
    │   ├── s2_completion.md
    │   ├── s3_completion.md
    │   └── s4_completion.md
    ├── evaluations/
    │   ├── s1_evaluation.md
    │   ├── s2_evaluation.md
    │   ├── s3_evaluation.md
    │   └── s4_evaluation.md
    ├── logs/
    │   └── harness.log
    └── summary.md
```

### Step 5: Summary (excerpt from .harness/notification-system/summary.md)

```markdown
# Harness Execution Summary

**Instance**: notification-system

| Sprint | Name | Score | Retries | Status |
|--------|------|-------|---------|--------|
| 1 | Notification Data Model & Storage | 8.1/10 | 0 | PASSED |
| 2 | Email Channel Integration | 7.3/10 | 1 | PASSED |
| 3 | In-App Channel & User Preferences | 7.6/10 | 0 | PASSED |
| 4 | Batching Engine & Delivery Pipeline | 7.0/10 | 0 | PASSED |

## Configuration
- Instance: notification-system
- Threshold: 7/10
- Eval Weights: functionality=30, reliability=30, compatibility=20, quality=20
- Total Duration: 30m 18s
```

## Output Structure

After execution, find all artifacts in `.harness/{name}/`:

```
{project_root}/.harness/
├── registry.json                   # Instance registry (tracks all runs)
└── {name}/                         # Instance directory
    ├── spec.md                     # Full specification
    ├── config.json                 # Harness configuration (includes sprints array)
    ├── contracts/
    │   └── s{N}_contract.md
    ├── completions/
    │   └── s{N}_completion.md
    ├── evaluations/
    │   └── s{N}_evaluation.md
    ├── logs/
    │   └── harness.log
    └── summary.md                  # Final report (generated in Step 4)
```

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Harness stops with "Spec file not found" | Spec file path doesn't exist | Verify path; use absolute path if needed |
| Sprint keeps failing after all retries | Criteria too strict or scope too large | Lower `--threshold`, increase `--max-retries`, or split the sprint |
| AUTO-SKIP cascades across most sprints | Early sprint failed with many dependents | Fix the failed sprint, then `--resume {name}` |
| Evaluator scores seem too low | Default eval-weights don't match priorities | Use `--eval-weights` to customize |
| Instance name conflict | Name already exists in registry | Use `--name` with a unique name, or let auto-naming append a suffix |
| Legacy .harness/ migration | Old flat structure detected | Auto-migrated on first command; check `.harness/default/` |
| "Instance not found" on resume | Typo in instance name | Run `--list` to see available instances |
| config.json shows "aborted" | Unrecoverable error during execution | Check `.harness/{name}/logs/harness-task.log`; fix and `--resume` |
