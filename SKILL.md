---
name: agent-fanout
description: Delegate implementation work to parallel headless agent-CLI subagents (codex by default; omp with a cheap model like Gemini Flash as an alternative) in persistent git worktrees, with the current agent as orchestrator (plan, review, merge, final QA). Use when the user asks to delegate to codex or omp, fan out work to subagents, or parallelize implementation across cheaper agents. Includes durable runner supervision and the recipe for subagents doing their own visual QA via dev-browser.
---

# Agent fan-out delegation

The current agent is the orchestrator: decompose, write precise briefs, review
every diff, merge, and own final verification. Headless runners implement. They
are cheaper and faster, not smarter—never merge their work unreviewed.

## Runtime boundary

Use `scripts/agent-fanout` for every run and lane lifecycle operation. It owns:

- canonical primary-worktree discovery, even when invoked from a linked worktree;
- machine-global run IDs and per-repository locks;
- integration and runner worktrees;
- durable supervision, state, PID/process-group cancellation, and recovery;
- per-run process isolation and cleanup.

The supervisor backend is an implementation detail. Do not call `tmux`, inspect
PIDs, use Claude `run_in_background`, or hand-build watcher loops. Normal
orchestration uses only:

```text
agent-fanout init / add-lane / start / status / wait / logs / collect / cancel / cleanup
```

`debug` and `attach` are maintainer/human escape hatches. Use them only when
ordinary status, logs, and collection cannot explain a lane.

## Runners

| Runner | Use | Sandbox |
|---|---|---|
| `codex` (default) | substantial phases needing judgment | workspace-write seatbelt |
| `omp` | mechanical/template phases, high-volume cheap work | none—full user permissions |
| `command` | testing or another headless CLI | whatever the command provides |

Codex lanes always run through the bundled progress watchdog. It enables
workspace-write network access by default, disables headless clarification
prompts, watches child-bound JSON/worktree/result progress, and fails dead or
quiet lanes with durable diagnostics.

OMP uses `--auto-approve`, so scope its brief tightly. It needs one-time auth
(`omp` then `/login`) or a provider key such as `GEMINI_API_KEY`.

## Architecture rules

1. **One integration worktree per run.** The controller creates it from the
   remote target branch (or clean local HEAD without a remote). Multiple runs
   on the same machine/repo remain isolated.
2. **Foundation first.** If phases need shared schema, types, or conventions,
   implement/review/commit that foundation on the integration branch before
   adding parallel lanes.
3. **Disjoint lane ownership.** Each phase owns explicit files/functions.
   Overlap means the decomposition failed; resolve it before launching.
4. **Persistent controller-owned worktrees.** Never combine harness-managed
   worktree isolation with a durable runner.
5. **Durable state is authoritative.** Completion notifications are not part
   of correctness. Use `status`, bounded `wait`, and `collect`.
6. **No eager interruption.** Analysis can leave a worktree unchanged for
   minutes. The watchdog owns liveness. Cancel only for user stop,
   destructive/out-of-scope writes, or a confirmed failure.
7. **The orchestrator owns review and final QA.** Runner tests/screenshots are
   a first-line filter, never sign-off.

## 0. Plan phases

- Identify any foundation phase that must land before parallel work.
- Give every parallel phase disjoint ownership.
- Choose Codex for judgment-heavy work and OMP for mechanical work.
- Define exact tests and visual criteria in every brief.

## 1. Initialize a durable run

From any worktree in the target repository:

```bash
AF=/absolute/path/to/agent-fanout/scripts/agent-fanout
$AF init --repo "$PWD"
```

Optional target override:

```bash
$AF init --repo "$PWD" --base-ref origin/release
```

Record the returned `run` ID and `integration_worktree`. Reuse the literal run
ID in later calls; shell variables do not persist between tool calls.

The controller atomically claims the run, finds the primary checkout, updates
the shared worktree exclusion under a repository lock, fetches the base, and
creates `int-<run>` under the primary checkout's `.worktrees/` directory.

If visual QA is needed, pre-warm dev-browser once from the orchestrator:

```bash
dev-browser <<< 'console.log("daemon warm")'
```

The daemon is machine-global. Prefix browser names with the run ID and allocate
unique dev-server ports.

## 2. Add and warm lane worktrees

```bash
$AF add-lane --run <run-id> --phase <phase>
```

The output gives the lane worktree. A fresh worktree has no gitignored inputs,
so warm what the repository actually needs before launch:

```bash
git -C <integration-worktree> ls-files -o -i --exclude-standard --directory
```

Clone copy-on-write inputs such as `node_modules`, `.venv`, `.env*`, generated
sources, and self-validating expensive compiler caches such as Cargo `target/`.
Skip outputs/ephemera (`dist`, `.next`, logs, screenshots) and never clone
`.worktrees/` recursively.

Use APFS/btrfs copy-on-write where available:

```bash
cp -cR <source> <lane-destination> 2>/dev/null \
  || cp -a --reflink=auto <source> <lane-destination> 2>/dev/null \
  || ln -s <source> <lane-destination>
```

Do not add `sccache` merely to share Cargo worktree builds: changing
`RUSTC_WRAPPER` changes fingerprints and can invalidate the cloned target cache.

## 3. Write briefs

Each brief must include:

```text
Goal and relevant context.

You are working in <absolute lane worktree>. Run pwd first. If it is not this
worktree, stop and report.

SCOPE—own only: <files/functions>. Do not touch anything else.
Do not commit. The orchestrator reviews and merges the working-tree diff.

Do not spin indefinitely. Implement once the local pattern is clear. If blocked
or no change is needed, report that conclusion and exit.

TEST PROCEDURE:
<exact commands>

REPORT: changes, tests, visual evidence, known gaps.
```

For visual QA, use a globally unique browser name `<run-id>-<phase>` and tell
the runner exactly what the screenshot must prove.

## 4. Start lanes

These are short foreground dispatch calls. Never set `run_in_background`.

Codex:

```bash
$AF start --run <run-id> --phase <phase> --brief <brief-file> --runner codex
```

Optional Codex knobs:

```bash
$AF start --run <run-id> --phase <phase> --brief <brief-file> \
  --runner codex --model <model> --reasoning-effort high
```

OMP:

```bash
$AF start --run <run-id> --phase <phase> --brief <brief-file> \
  --runner omp --model gemini-3.5-flash --thinking low
```

Another headless command:

```bash
$AF start --run <run-id> --phase <phase> --brief <brief-file> \
  --runner command -- <command> <args...>
```

## 5. Observe and recover semantically

```bash
$AF status --run <run-id>
$AF status --run <run-id> --json
$AF wait --run <run-id> --phase <phase> --timeout 60
$AF logs --run <run-id> --phase <phase> --tail 80
$AF logs --run <run-id> --phase <phase> --events --tail 80
$AF collect --run <run-id> --phase <phase>
```

Lane states:

```text
created → dispatching → running → succeeded | failed | cancelled | interrupted
```

- `running`: keep waiting; unchanged worktree is not failure.
- `succeeded`: inspect the report and complete diff.
- `failed`: read result/log/events; salvage only complete, correct work.
- `interrupted`: the supervisor vanished before a terminal result. Inspect the
  durable worktree, then relaunch or take the work back explicitly.
- `cancelled`: explicit stop completed and the child process group is gone.

After an orchestrator/session restart, call `status` with the recorded run ID.
Live lanes remain live; completed lanes retain their results; missing
nonterminal lanes become `interrupted`.

To stop an authorized lane:

```bash
$AF cancel --run <run-id> --phase <phase>
```

Do not send raw signals or kill backend sessions.

## 6. Review and apply each lane

`collect` prints the lane worktree and result path. Capture tracked and new
files by staging in the runner tree, then review the index diff:

```bash
git -C <lane-worktree> add -A
git -C <lane-worktree> diff --cached > <run-namespaced-patch>
```

Read every hunk and the runner report. A plain `git diff` is insufficient
because it omits untracked new files.

Apply accepted work to the integration tree:

```bash
git -C <integration-worktree> apply --3way --index <patch>
```

Re-run relevant checks after every applied phase. Empty patch is success only
when the report explains why no change was needed and the orchestrator agrees.

## 7. Final QA and landing

On the integration branch, run the repository's full local gate: formatting,
lint, typecheck/build, unit/integration tests, and the actual e2e command from
package scripts or CI. Re-check important UI surfaces yourself where relevant.

Commit only after the combined tree is green. With a remote, push the unique
integration branch, open a PR, wait for required checks, then merge according
to the repository's policy. Without a remote, use the repository's local
landing policy; concurrent local landings must be serialized and rebased when
the target advanced.

## 8. Cleanup

After work is collected and safely landed:

```bash
$AF cleanup --run <run-id> --drop-worktrees --force
```

`--force` is an explicit acknowledgment that controller-owned worktrees and
branches may contain reviewed/staged runner changes. Cleanup is scoped to the
run and cannot terminate another run's supervisor.

Durable reports remain in the state directory for diagnosis. Use `list` to
find known runs:

```bash
$AF list
```

## Failure reference

| Symptom | Meaning | Action |
|---|---|---|
| `failed`, exit 124 | Codex dead-started or went quiet | inspect result/log/events; relaunch once or take back |
| `interrupted` | supervisor/process disappeared without terminal state | inspect worktree; salvage or relaunch explicitly |
| `running` with no diff | normal analysis/setup unless watchdog later fails it | wait; do not interrupt |
| browser daemon timeout | daemon was not pre-warmed outside sandbox | pre-warm, then relaunch |
| missing deps/toolchain | worktree was not warmed | warm inputs/cache; relaunch once |
| apply conflict | phase ownership overlapped or integration advanced | resolve/review manually; do not blindly re-delegate |
| cleanup refuses | live lane or missing explicit destructive acknowledgment | collect/cancel first; then use documented cleanup |

The orchestrator reasons about durable lanes and Git patches—not backend
processes. If ordinary commands cannot explain state, use `debug`; reserve
`attach` for direct human investigation.
