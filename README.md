# agent-fanout

Delegate implementation work to parallel headless agent-CLI subagents — codex by default, or omp with a cheap model like Gemini Flash — in persistent git worktrees, with the orchestrating agent planning the phases, writing the briefs, reviewing every diff, merging, and owning final verification.

## Install

```bash
npx skills add gbasin/agent-fanout --all -g
```

## What it does

The runners do the volume work — cheaper and faster, reviewed before anything lands. The skill encodes the working recipe for orchestrating them:

1. **The orchestrator runs in its own integration worktree** — never the primary checkout, so several orchestrators can fan out on one machine/repo at once; every branch, `.worktrees/` path, and `/tmp` artifact is session-namespaced, and main is touched only by the final atomic merge/PR
2. **Foundation first** — if the work needs shared conventions, run one task, review, commit, then fan out
3. **Disjoint ownership** — each parallel phase owns specific files or functions; shared-file additions go in marked appendix blocks so merges stay trivial
4. **Persistent worktrees, launched through a Codex startup watchdog** — completion is still process exit, but a dead-started Codex lane fails fast instead of sitting alive for hours with only `task_started`
5. **No status-polling watcher loops** — the launch wrapper supervises startup liveness; the background-task notification remains the "done/failed" signal
6. **Patch-based review and merge** — read every diff, `git apply --3way`, re-test after each apply
7. **Final QA belongs to the orchestrator** — runner screenshots are a first-line filter, not a sign-off; then land the integration branch (PR if the repo has a remote, else local merge)

Mixed fleets are fine: codex for phases needing judgment, omp + Gemini Flash for mechanical ones. Any agent CLI that runs headless, exits on completion, and works against the current directory can slot in.

## Why it's shaped this way

Distilled postmortem — a real orchestration session hit every failure mode at least once:

- Harness-managed worktree isolation reaped the worktree out from under a still-running async codex task, orphaning it in a deleted directory
- Job-status watchers parsed "no job found" (per-directory job state, checked from the wrong directory) as "finished"
- A codex task wedged before its first real event: the process was alive with only `session_meta` + `task_started`, no result file, and no worktree diff
- A codex task wedged for ~2 hours *after* completing its work, retrying browser fallbacks its sandbox could never satisfy
- Embedded worktree gitlinks snuck into a commit via `git add -A`; the procedure excludes `.worktrees/` before creating nested worktrees

## The visual QA trick

Headless codex under the macOS seatbelt sandbox *cannot launch* a browser (Chrome dies at Mach-port registration). But it can *drive* one: pre-start the [dev-browser](https://github.com/gbasin/dev-browser) daemon outside the sandbox and launch the lane through `scripts/launch-codex-lane`, which passes the required network flag by default:

```bash
scripts/launch-codex-lane --worktree "$WT" --brief "$BRIEF" --result "$RESULT" --run-log "$RUN_LOG"
```

The socket connect succeeds, screenshots are written daemon-side, and codex reads the PNGs back to visually verify its own work. Both halves are required — without the network flag the connect is blocked even with the daemon running. Unsandboxed runners (omp) can drive the daemon directly, but pre-starting it still avoids auto-start races between parallel tasks.

## Requirements

- At least one runner CLI with auth configured: Codex CLI (`codex exec`) and/or omp (`omp /login` once, or a provider key such as `GEMINI_API_KEY`)
- Optional, for visual QA: [dev-browser](https://github.com/gbasin/dev-browser)

## License

[MIT](LICENSE)
