---
name: codex-fanout
description: Delegate implementation work to parallel codex CLI subagents in persistent git worktrees, with Claude as orchestrator (plan, review, merge, final QA). Use when the user asks to delegate to codex, fan out work to codex subagents, or parallelize implementation across cheaper agents. Includes the recipe for codex doing its own visual QA via dev-browser.
---

# Codex fan-out delegation

Claude is the orchestrator: decompose, write precise briefs, review every diff,
merge, and own final verification. Codex implements. It is cheaper and faster,
not smarter — never merge its work unreviewed.

## Hard-won architecture rules

1. **Never combine the Agent tool's `isolation: worktree` with codex.** The
   codex-rescue launcher starts the task async and exits in ~60s; the harness
   then reaps the "unchanged" worktree, orphaning the still-running codex task
   in a deleted directory. Create persistent worktrees manually (below).
2. **Use `codex exec` directly, not `codex-companion.mjs` background tasks,
   for fan-out.** Companion job state is keyed per-cwd: status checks from any
   other directory return "No job found", which grep-based watchers misread as
   "finished". `codex exec` completion = process exit = automatic Bash
   task notification. No polling loops, no false positives.
3. **Always pass `--sandbox workspace-write` explicitly.** The global
   `~/.codex/config.toml` may set `sandbox_mode = "danger-full-access"`; a bare
   `codex exec` would then run unsandboxed.
4. **Codex CAN do visual QA** (verified 2026-06-12), but only if BOTH hold:
   - the dev-browser daemon was pre-started from outside the sandbox
     (daemon auto-start inside seatbelt always fails at Mach-port registration,
     and the task may then wedge for hours retrying fallbacks);
   - codex runs with `-c 'sandbox_workspace_write.network_access=true'`
     (otherwise the connect to `~/.dev-browser/daemon.sock` is blocked and
     dev-browser reports "Daemon failed to start within 5 seconds").
   Screenshot writes happen daemon-side into `~/.dev-browser/tmp/`, which codex
   can read and view as images — so it can check its own work visually.

## Procedure

### 0. Plan phases
- If the work needs shared conventions (design system, schema, API shape), run
  a **foundation phase first as a single task**, review and commit it, THEN fan
  out. Parallel phases inherit the committed foundation.
- Give parallel phases **disjoint ownership**: each owns specific files or
  specific functions. Additions to shared files go in a clearly marked
  appendix block (`# === <phase> additions ===`) to keep merges trivial.

### 1. Pre-flight (orchestrator, main repo)
```bash
# avoid the gitlink trap: embedded worktrees must never reach `git add -A`
grep -qx '.claude/worktrees/' .git/info/exclude 2>/dev/null \
  || echo '.claude/worktrees/' >> .git/info/exclude

# if any phase needs visual QA: warm the daemon OUTSIDE the sandbox
dev-browser <<< 'console.log("daemon warm")'
```
Start from a clean, committed state — codex diffs are reviewed against HEAD.

### 2. Persistent worktrees (one per parallel phase)
```bash
git worktree add .claude/worktrees/<phase> -b wt-<phase>
```

### 3. Briefs
Write each brief to `/tmp/<phase>-brief.md` (avoids shell quoting issues).
Template:

```
<one-paragraph goal and context>

You are working in <ABSOLUTE worktree path>. Run `pwd` first to confirm.
If this directory does not exist or is not a git worktree, STOP and report —
do not work anywhere else.

SCOPE — you own ONLY: <files / functions>. Additions to shared files go in an
appendix block marked `# === <phase> additions ===`. Touch nothing else.

Do NOT commit. The orchestrator reviews and merges your working-tree diff.

TEST PROCEDURE (run before reporting):
<exact build/test commands>
# visual check (daemon is already running; use a unique browser name):
cat > /tmp/<phase>-shot.js <<'JS'
const page = await browser.getPage("<phase>");
await page.setViewportSize({ width: 1440, height: 1000 });
await page.goto("file://<path under the worktree>");
await page.waitForTimeout(300);
console.log(await saveScreenshot(await page.screenshot({ fullPage: true }), "<phase>.png"));
JS
dev-browser --browser <phase> run /tmp/<phase>-shot.js
# then VIEW the saved png and verify: <concrete visual criteria>

REPORT: what changed, how you tested (including what the screenshot showed),
known gaps.
```

### 4. Launch (one backgrounded Bash call per phase)
```bash
cd <abs worktree path> && codex exec \
  --sandbox workspace-write \
  -c 'sandbox_workspace_write.network_access=true' \
  -o /tmp/<phase>-result.md \
  "$(cat /tmp/<phase>-brief.md)"
```
Run with `run_in_background: true`. The completion notification fires when the
process exits — do not write watcher loops. Knobs for hard phases:
`-m <model>`, `-c model_reasoning_effort="high"`.

### 5. Monitor (only when prompted or suspicious)
Tail the Bash output file. If no new output for ~10 minutes, check
`git -C <worktree> diff --stat` — codex sometimes wedges AFTER the work is
done (the Phase-1 failure mode). If the diff looks complete, kill the process
and salvage the tree; the work is rarely lost.

### 6. Review and merge (orchestrator)
```bash
git -C .claude/worktrees/<phase> diff > /tmp/<phase>.patch
# READ the patch, check the result file, then:
git apply --3way /tmp/<phase>.patch
```
Re-run build/tests on main after each apply. Overlapping hunks across phases
mean the ownership split failed — resolve manually, don't re-delegate.

### 7. Final QA — yours, not codex's
Codex screenshots are a first-line filter. Before committing, re-verify the key
surfaces yourself (desktop + mobile widths at minimum). Then commit and:
```bash
git worktree remove .claude/worktrees/<phase> --force
git branch -D wt-<phase>
```

## Failure-mode quick reference

| Symptom | Cause | Action |
|---|---|---|
| codex: "Daemon failed to start within 5 seconds" | daemon not pre-warmed, or `network_access` not set | warm daemon from orchestrator; relaunch with the `-c` flag |
| codex stuck retrying browser fallbacks (node_repl, NODE_PATH, --connect) | same as above | same; salvage any completed diff first |
| worktree vanished mid-task | it was harness-managed (Agent isolation) | kill orphans (`pkill -f app-server-broker` for that cwd may need escalation), recreate persistent worktree, relaunch |
| companion `status`: "No job found" | per-cwd state; checked from wrong dir | `cd` to the exact launch dir; never treat as completion |
| task "running" for an hour with frozen progress tail but full diff present | end-of-run wedge | kill, salvage tree, do verification yourself |
| stray gitlinks in a commit | `.claude/worktrees/` not excluded | pre-flight exclude line; fix commit with `git rm --cached` |
