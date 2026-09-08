# Shared validation resources

Use this procedure when local builds, browsers, databases, or simulators contend
for one machine. Implementation concurrency and validation concurrency are
separate budgets. Start with two implementation lanes on a constrained host.
Keep read-only reviews bounded too. Tune from measured load and useful throughput.

## Resource ownership

Run heavy validation through `scripts/resource-run`. It provides one `heavy`
slot, two `implementation` slots for foreground commands, and one slot per
`simulator-<device-id>`. Locks are shared across runs and repositories for the
current OS user. Different OS users need a shared service or a dedicated host.
It does not discover or throttle existing processes, nor limit lanes launched
without it. Limit implementation dispatches in the orchestrator. Never wrap a
short `agent-fanout start` dispatch in a slot and assume the lane remains limited.

```bash
scripts/resource-run --resource heavy --resource simulator-<device-id> -- \
  scripts/validate-run --job /absolute/path/job.json --output /absolute/path/new-evidence-dir
```

For a durable job, put this command inside a controller `--runner command` lane.
Use the controller for its lifecycle and bounded waits. Resource waits are not
application hangs. Do not wrap controller waits in another background task.

`resource-run` accepts `--timeout SECONDS` for slot acquisition. Commands run in
an owned process group. Cancellation terminates that group. Background children
in the group are terminated when the command exits. Keep owned servers in the
foreground under a supervisor for the duration of the job. Do not daemonize them
into separate sessions. Unrelated processes are never targeted by name.

All participating sessions must use the same lock directory. Its default is
`~/.local/state/agent-fanout/resources`, independent of per-run state overrides.
`AGENT_FANOUT_RESOURCE_HOME` is an isolation override for tests or an explicitly
coordinated installation. Changing it per run defeats machine-wide exclusion.

## Evidence and ordered steps

`validate-run` accepts a JSON job. Commands are argv arrays, executed without a
shell. Paths must be absolute or relative to `worktree`. The evidence directory
must be new and outside the worktree. Store no secrets in the job or command
output. `environment` should contain explicit public build configuration.

```json
{
  "worktree": "/absolute/path/validation-worktree",
  "commit": "full-commit-sha",
  "environment": {"EXPO_PUBLIC_IKE_FIXTURES": "1"},
  "step_timeout": 1800,
  "build": ["./tools/build-fixture-app"],
  "verify": ["./tools/verify-installed-fixture-app"],
  "health": ["./tools/check-owned-simulator"],
  "smoke": ["maestro", "test", "apps/agent/maestro/smoke.yaml"],
  "test": ["maestro", "test", "apps/agent/maestro/affected-flow.yaml"],
  "collect": ["./tools/collect-simulator-failure"],
  "recover": ["./tools/restart-owned-simulator"]
}
```

The `tools/*` paths above are application-specific adapters, not bundled tools.
Use actual commands from the repository. Inspect their behavior before use.
For browser or package validation, omit simulator adapters. `smoke` and `test`
are required. `build`, `verify`, `health`, `collect`, and `recover` are optional,
except that building or reusing an artifact requires `verify`.

The helper requires a clean Git worktree. It checks the commit and dirty state
before and after each step. Reserve that checkout exclusively for validation.
These checks detect ordinary drift but cannot prevent a concurrent edit that is
made and reverted between checks. Generated outputs must follow repo ignore rules.

A failed build stops before verification or tests. `verify` must check that the
installed artifact matches the expected source and configuration, not merely
that an app with the same bundle identifier exists. The helper records commit,
configuration digest, stage outcomes, and logs. It cannot infer installation
identity without the repository's verification adapter.

Reuse an artifact with `--reuse-build /path/to/previous/result.json` only when its
receipt matches the source, build command, verification command, and explicit
configuration. The installation is verified again before tests. Flow-only edits
may reuse a build when the application's build inputs are unchanged, but the
generic helper conservatively invalidates its receipt on any commit change.
Repository tooling may supply a more precise build-input fingerprint. Do not
reuse on the basis of a bundle ID or successful prior build alone.

## Simulator failures

One QA owner holds the device lease. Other lanes submit affected flows and read
the resulting screenshots and accessibility trees. Prove one representative
fixture flow before expanding the suite. Prefer the project's stable fixture
Release build over development-client prompt dismissal loops.

`health` must check the target simulator and detect new simulator crash reports
since this run began, including SpringBoard crashes that have already restarted.
A failed UI assertion alone is not evidence of an infrastructure crash.
`collect` preserves Maestro artifacts and matching simulator reports before
recovery. `recover` may restart only the leased simulator. It must not erase
devices or terminate other runs.

The helper attempts infrastructure recovery at most once. It verifies the
artifact again and reruns the smoke flow before affected tests. A second
infrastructure failure terminates the job. Product test failures are returned
without automatic retries or source edits. Run the full regression after the
affected flows pass and the combined source is fixed.

Browser sessions also need separate test data. Allocate distinct fixture data
and ports for independent tests. Coordinate shared state intentionally for a
multi-actor journey. Start Docker and browser services only when needed. Release
owned services when their last dependent job finishes.
