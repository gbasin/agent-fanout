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
same phase names—without sharing supervisor namespaces. Heavy commands can share
per-user machine-wide slots through
`scripts/resource-run`. `scripts/validate-run` records fixed-source evidence and
runs build, verification, smoke, and test steps in order. See
[validation resources](references/validation.md). These opt-in wrappers do not
throttle existing processes or automatically cap controller lane dispatches.
Callers still budget implementation lanes, API quota, and provider rate limits.

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

For pnpm lanes, bootstrap on the host with a shared store outside all worktrees,
then pass `--pnpm-store /absolute/path/to/store` to `start --runner codex`.
The launcher checks store selection and write access inside the sandbox before
starting the agent. It grants access only to the resolved store directory and
pins the store in agent shell commands. A failed check leaves the lane failed
with diagnostics in its run log. The store must already exist; this option
does not install dependencies or migrate existing worktrees. Share it only
among mutually trusted jobs. Each worktree retains its own `node_modules`.

This grants the package store, not pnpm's metadata cache or Corepack's cache.
Bootstrap the repository's pinned pnpm version and dependencies on the host.
If a lane changes dependencies and needs cache writes outside the store,
return to host bootstrap before continuing.

The preflight uses `codex sandbox` writable roots; the agent receives
`codex exec --add-dir`. The smoke test checks the sandbox and real installs;
the launcher tests check the arguments without running a model. Recheck this
boundary after a Codex upgrade.

The option requires Python 3, pnpm, and Codex's `sandbox` helper. Other lanes
keep their existing permissions. See [store settings](https://pnpm.io/settings/store)
and [Codex directory access](https://developers.openai.com/codex/cli/reference).

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
python3 scripts/test-pnpm-store
python3 scripts/test-validation
```

For an opt-in macOS smoke test with real Codex sandboxing and pnpm, run
`python3 scripts/test-pnpm-store-sandbox`. It installs a local fixture offline
in two disposable worktrees and checks that unrelated writes stay blocked.
It runs no model. Set `FANOUT_TEST_PNPM_BIN` to an installed pnpm executable to test
a specific version without downloading it.

The sandbox smoke test has passed with pnpm 9.10.0, 11.15.1, 11.21.0,
12.0.0, and 12.2.0. After a pnpm upgrade, warm the store on the host with the new
version before launching lanes. The preflight resolves the versioned store
path with `pnpm store path`; it does not hard-code a store version.

The controller suite uses disposable repositories and exercises concurrent
initialization, linked-worktree discovery, duplicate run rejection, same-named
lanes in independent runs, durable failure, forced cancellation, supervisor
loss, and cleanup isolation.

## Requirements

- Git, tmux, Perl, and standard macOS/Linux command-line tools
- Python 3 for the optional resource and validation helpers
- At least one authenticated runner: Codex CLI and/or OMP (`omp /login`, or a
  provider key such as `GEMINI_API_KEY`)
- Optional for visual QA: [dev-browser](https://github.com/gbasin/dev-browser)

See [SKILL.md](SKILL.md) for the complete orchestration and review workflow.

## License

[MIT](LICENSE)
