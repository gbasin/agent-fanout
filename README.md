# codex-fanout

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for
delegating implementation work to parallel [Codex
CLI](https://github.com/openai/codex) subagents, with Claude as the
orchestrator: it plans the phases, writes the briefs, reviews every diff,
merges, and owns final verification. Codex does the volume work — cheaper and
faster, reviewed before anything lands.

## Why this exists

This skill is a distilled postmortem. A real orchestration session hit every
failure mode in this workflow at least once:

- Harness-managed worktree isolation reaped the worktree out from under a
  still-running async codex task, orphaning it in a deleted directory.
- Job-status watchers parsed "no job found" (per-directory job state, checked
  from the wrong directory) as "finished".
- A codex task wedged for ~2 hours *after* completing its work, retrying
  browser fallbacks its sandbox could never satisfy.
- Embedded worktree gitlinks snuck into a commit via `git add -A`.

The skill encodes the working recipe instead: persistent manually-created
worktrees, `codex exec` with process-exit completion semantics (no watcher
loops), disjoint phase ownership, patch-based review and merge, and a tested
configuration that lets sandboxed codex do its own visual QA.

## The visual QA trick

Headless codex under the macOS seatbelt sandbox *cannot launch* a browser
(Chrome dies at Mach-port registration). But it can *drive* one: if the
`dev-browser` daemon is pre-started outside the sandbox and codex runs with

```
codex exec --sandbox workspace-write -c 'sandbox_workspace_write.network_access=true' ...
```

then the socket connect succeeds, screenshots are written daemon-side, and
codex can read the PNGs back and visually verify its own work. Both halves are
required — without the network flag the connect is blocked even with the
daemon running.

## Requirements

- Claude Code (the skill host / orchestrator)
- Codex CLI (`codex exec`) with auth configured
- `git` with worktree support (any modern git)
- Optional, for visual QA: the `dev-browser` CLI and daemon

## Install

```bash
git clone https://github.com/gbasin/codex-fanout ~/.claude/skills/codex-fanout
```

Claude Code picks it up automatically; ask Claude to "fan out this work to
codex" or invoke `/codex-fanout`.

## License

MIT
