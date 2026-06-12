# agent-fanout

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for
delegating implementation work to parallel headless agent-CLI subagents, with
Claude as the orchestrator: it plans the phases, writes the briefs, reviews
every diff, merges, and owns final verification. The runners do the volume
work — cheaper and faster, reviewed before anything lands.

Supported runners (mixed fleets are fine, assigned per phase):

- **[Codex CLI](https://github.com/openai/codex)** (default) —
  `codex exec` under the macOS seatbelt sandbox; best for phases needing
  judgment.
- **omp (oh-my-pi)** with a cheap model like Gemini Flash —
  `omp -p --auto-approve --model gemini-3.5-flash`; unsandboxed, fastest and
  cheapest, best for mechanical phases.

Any agent CLI that runs headless, exits on completion, and works against the
current directory can slot in the same way.

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
worktrees, headless runs with process-exit completion semantics (no watcher
loops), disjoint phase ownership, patch-based review and merge, and a tested
configuration that lets sandboxed runners do their own visual QA.

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
daemon running. Unsandboxed runners (omp) can drive the daemon directly, but
pre-starting it from the orchestrator still avoids concurrent auto-start races
between parallel tasks.

## Requirements

- Claude Code (the skill host / orchestrator)
- At least one runner CLI with auth configured:
  - Codex CLI (`codex exec`), and/or
  - omp (`omp /login` once, or a provider key such as `GEMINI_API_KEY`)
- `git` with worktree support (any modern git)
- Optional, for visual QA: the `dev-browser` CLI and daemon

## Install

```bash
git clone https://github.com/gbasin/agent-fanout ~/.claude/skills/agent-fanout
```

Claude Code picks it up automatically; ask Claude to "fan out this work to
codex" (or omp) or invoke `/agent-fanout`.

## License

MIT
