# agent-fanout

Delegate implementation work to parallel headless agent CLIs in persistent Git
worktrees. The orchestrating agent plans the phases, reviews every diff, merges,
and owns final verification.

## Install

```bash
npx skills add gbasin/agent-fanout --all -g
```

## Durable controller

All lane lifecycle operations go through `scripts/agent-fanout`:

```bash
AF=/absolute/path/to/agent-fanout/scripts/agent-fanout

$AF init --repo /path/to/repo
$AF add-lane --run <run-id> --phase api
$AF start --run <run-id> --phase api --brief /path/to/api-brief.md --runner codex
$AF status --run <run-id>
$AF wait --run <run-id> --phase api --timeout 60
$AF collect --run <run-id> --phase api
```

The controller owns the process backend, per-run isolation, worktree creation,
durable state, process-group cancellation, and cleanup. Orchestrators work with
semantic states—`created`, `dispatching`, `running`, `succeeded`, `failed`,
`cancelled`, and `interrupted`—rather than tmux sessions or PIDs. `debug` and
`attach` exist only as maintainer/human escape hatches.

This avoids relying on a host harness's background-task lifetime. Starting a
lane is a short foreground dispatch; after that, the controller-owned supervisor
survives the orchestrator process and records terminal state on disk.

## Isolation and concurrency

- Every run has a globally unique ID, its own integration worktree, and its own
  supervisor server.
- Every lane has a distinct branch, worktree, state directory, and process group.
- Short shared Git mutations are serialized with a per-repository lock.
- Cancellation validates recorded process identity before targeting the lane's
  process group.
- Cleanup is run-scoped and refuses live lanes unless explicitly forced.

Separate orchestrators can therefore fan out in the same repository—or use the
same phase names—without sharing supervisor namespaces. There is intentionally
no machine-wide concurrency limit; callers remain responsible for CPU, memory,
API quota, and provider rate limits.

## Runners

- `codex` (default): launches `codex exec` through the bundled progress watchdog.
- `omp`: launches `omp -p --no-session --auto-approve`; scope briefs tightly
  because it is unsandboxed.
- `command`: supervises an arbitrary headless command, useful for testing or
  integrating another agent CLI.

Codex lanes enable workspace-write network access by default and detect dead
starts or quiet wedges from the child JSON stream, result file, and worktree
progress. Completion notifications are not part of correctness; use `status`,
bounded `wait`, and `collect` after a restart.

## Visual QA

Headless Codex under the macOS seatbelt cannot launch Chrome, but it can drive a
pre-started [dev-browser](https://github.com/gbasin/dev-browser) daemon. Warm the
daemon from the orchestrator, give each lane a run-prefixed browser name and a
unique port, then launch the lane through the controller. The Codex launcher
passes the required network flag by default.

## Tests

```bash
scripts/test-agent-fanout
scripts/test-launch-codex-lane
```

The controller suite uses disposable repositories and exercises concurrent
initialization, linked-worktree discovery, duplicate run rejection, same-named
lanes in independent runs, durable failure, forced cancellation, supervisor
loss, and cleanup isolation.

## Requirements

- Git, tmux, Perl, and standard macOS/Linux command-line tools
- At least one authenticated runner: Codex CLI and/or OMP (`omp /login`, or a
  provider key such as `GEMINI_API_KEY`)
- Optional for visual QA: [dev-browser](https://github.com/gbasin/dev-browser)

See [SKILL.md](SKILL.md) for the complete orchestration and review workflow.

## License

[MIT](LICENSE)
