# harness-task

`harness-task` is a Codex/Claude-style skill for running large implementation work as a structured multi-sprint workflow.

It applies the Harness methodology with strict Generator/Evaluator separation, file-based handoff, retry loops, and multi-instance state management under `.harness/`.

## Features

- Run from an existing spec file or from a plain-language task description
- Auto-plan and decompose work into sequential sprints
- Separate Generator and Evaluator phases to reduce self-evaluation bias
- Retry failed sprints with structured fix feedback
- Support multiple concurrent harness instances in one project
- Resume interrupted runs and archive completed ones
- Track execution state in files so progress is inspectable and recoverable

## Repository Contents

- `SKILL.md`: executable skill definition
- `examples.md`: usage examples and common scenarios
- `methodology.md`: background on the Harness methodology
- `scripts/`: reserved for helper scripts

## Usage

Example with an existing spec:

```bash
/harness-task docs/spec.md --name auth-refactor --threshold 8 --max-retries 3
```

Example from a description:

```bash
/harness-task "Refactor the authentication system from session-based to JWT while maintaining backward compatibility"
```

Management commands:

```bash
/harness-task --list
/harness-task --resume auth-refactor
/harness-task --archive auth-refactor
```

## Key Arguments

- `--name <name>`: set the harness instance name
- `--threshold <1-10>`: evaluation pass threshold
- `--max-retries <N>`: retry count per sprint
- `--start-from <N>`: start from a specific sprint
- `--skip-eval`: skip the evaluator phase
- `--sprints <N>`: override sprint count
- `--eval-weights '{...}'`: customize scoring weights
- `--list`: list all harness instances
- `--resume <name>`: resume an existing instance
- `--archive <name>`: archive a completed or aborted instance

## Output Structure

Harness state is stored in `.harness/` at the project root:

```text
.harness/
├── registry.json
└── <instance-name>/
    ├── spec.md
    ├── config.json
    ├── contracts/
    ├── completions/
    ├── evaluations/
    ├── logs/
    └── summary.md
```

## Notes

- All generated output should follow the language of the provided spec
- The Generator and Evaluator are intentionally isolated
- Multiple harness instances can coexist in the same repository

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
