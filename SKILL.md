---
name: agent-fanout
description: Delegate implementation work to parallel headless agent-CLI subagents (codex by default; omp with a cheap model like Gemini Flash as an alternative) in persistent git worktrees, with Claude as orchestrator (plan, review, merge, final QA). Use when the user asks to delegate to codex or omp, fan out work to subagents, or parallelize implementation across cheaper agents. Includes the recipe for subagents doing their own visual QA via dev-browser.
---

# Agent fan-out delegation

Claude is the orchestrator: decompose, write precise briefs, review every diff,
merge, and own final verification. Headless agent CLIs implement. They are
cheaper and faster, not smarter — never merge their work unreviewed.

## Runners

Any agent CLI qualifies as a runner if it (a) runs headless and **exits when
the task is done** (completion = process exit, which fires the Bash background
notification automatically), (b) operates on the current working directory,
and (c) follows a "do not commit" instruction. Pick per phase; mixed fleets
are fine.

| Runner | Launch core | Best for | Sandbox |
|---|---|---|---|
| **codex** (default) | `codex exec --sandbox workspace-write ...` | substantial phases needing judgment | macOS seatbelt; reads everywhere, writes workspace+/tmp |
| **omp** (oh-my-pi) | `omp -p --no-session --auto-approve --model gemini-3.5-flash ...` | mechanical/template phases, high-volume cheap work | **none** — full user permissions |

Runner notes:
- **codex**: ALWAYS pass `--sandbox workspace-write` explicitly — the global
  `~/.codex/config.toml` may set `danger-full-access`, and a bare `codex exec`
  would run unsandboxed. ALSO ALWAYS pass
  `-c 'features.default_mode_request_user_input=false'` — if the global config
  enables that under-dev feature, a headless lane that decides to ask a
  clarifying question blocks forever on stdin (no one answers), showing up as a
  task that "runs" for many minutes with zero output and zero diff. Knobs:
  `-m <model>`, `-c model_reasoning_effort="high"`.
- **omp**: `--auto-approve` means it can touch anything the user can; the
  worktree is discipline, not containment — keep briefs tightly scoped.
  Model is fuzzy-matched (`--model gemini-3.5-flash`, `--model flash`);
  `--thinking low|medium|high` trades speed for depth. Needs one-time auth
  (`omp` then `/login`, or a provider key like `GEMINI_API_KEY` in env) —
  headless runs fail fast with "Use /login, set an API key..." if missing.
  Briefs attach via `@/path/to/brief.md`; capture stdout as the result.

## Hard-won architecture rules

1. **The orchestrator works in its OWN integration worktree — never the
   primary checkout.** Multiple orchestrators may run on one machine/repo at
   once; the main checkout is a shared resource (merge target, test runner)
   and global branch/worktree names collide. Each run picks one short unique
   `<sid>`, creates `.claude/worktrees/_int-<sid>` on branch `int-<sid>`, and
   does ALL "main repo" work there. Runner worktrees branch off `int-<sid>`;
   main is touched only by the final atomic merge/PR (Step 7). Every branch,
   worktree path, and `/tmp` artifact is `<sid>`-namespaced so concurrent runs
   never collide.
2. **Never combine the Agent tool's `isolation: worktree` with an async
   runner.** If the launching subagent exits before the runner finishes, the
   harness reaps the "unchanged" worktree and orphans the still-running task
   in a deleted directory. Create persistent worktrees manually (below).
3. **Launch runners directly via backgrounded Bash; completion = process
   exit.** No status-polling watcher loops. (History: codex-companion
   background jobs key their state per-cwd, so `status` from any other
   directory says "No job found" — which grep watchers misread as "finished".)
4. **Visual QA from inside a runner works, with preconditions.** Pre-start the
   dev-browser daemon from the orchestrator before launching (avoids
   concurrent auto-start races; and under codex's seatbelt, daemon auto-start
   is impossible — Chrome dies at Mach-port registration, and the task may
   wedge for hours retrying fallbacks). codex additionally needs
   `-c 'sandbox_workspace_write.network_access=true'` or the connect to
   `~/.dev-browser/daemon.sock` is blocked ("Daemon failed to start within 5
   seconds"). Screenshot writes happen daemon-side into `~/.dev-browser/tmp/`,
   readable by all runners — they can view the PNGs and check their own work.

## Procedure

**Shell-state caveat — read first.** Each code block below runs as its own Bash
call. In this harness only the *working directory* persists between calls — NOT
shell variables or functions. So `<sid>` and `<phase>` are literal placeholders
you substitute (pick `<sid>` ONCE), and every block that needs them re-establishes
its vars and re-defines `warm()` at the top. Start any post-Step-1 block with this
preamble (works from the primary checkout or any worktree):
```bash
SID=<sid>                                                       # the token chosen in Step 1
MAIN=$(git worktree list --porcelain | awk 'NR==1{sub(/^worktree /,"");print}')
INT=$MAIN/.claude/worktrees/_int-$SID
```

### 0. Plan phases
- If the work needs shared conventions (design system, schema, API shape), run
  a **foundation phase first as a single task**, review and commit it, THEN fan
  out. Parallel phases inherit the committed foundation.
- Give parallel phases **disjoint ownership**: each owns specific files or
  specific functions. Additions to shared files go in a clearly marked
  appendix block (`# === <phase> additions ===`) to keep merges trivial.
- Assign runners by difficulty: codex for phases needing judgment, omp+flash
  for mechanical ones.

### 1. Pre-flight (orchestrator) — claim an integration worktree
```bash
MAIN=$(git rev-parse --show-toplevel); cd "$MAIN"
SID=<short-unique-token>          # pick ONCE (e.g. 2026-06-21-a3f); reuse literally everywhere

# avoid the gitlink trap: embedded worktrees must never reach `git add -A`
# (.git/info/exclude lives in the shared common dir — once covers all worktrees)
grep -qx '.claude/worktrees/' .git/info/exclude 2>/dev/null \
  || echo '.claude/worktrees/' >> .git/info/exclude

# the orchestrator's OWN isolated checkout; the primary checkout is never mutated
# until Step 7. Other orchestrators get their own _int-<their-sid> — no contention.
git worktree add .claude/worktrees/_int-$SID -b int-$SID
INT=$MAIN/.claude/worktrees/_int-$SID

# warm the integration tree with gitignored inputs (deps/env) so it can build,
# test, and run the foundation phase. Same warm() used in Step 2; define it here too
# (functions don't persist between Bash calls). Classify inputs per Step 2.
warm() {  # warm <src-root> <relpath>
  local src=$1/$2 dst=$INT/$2
  [ -e "$src" ] || return 0
  mkdir -p "$(dirname "$dst")"
  cp -cR "$src" "$dst" 2>/dev/null || cp -a --reflink=auto "$src" "$dst" 2>/dev/null \
    || ln -s "$src" "$dst"
}
warm "$MAIN" node_modules; warm "$MAIN" .env        # the classified inputs

# if any phase needs visual QA: warm the daemon OUTSIDE any sandbox. The daemon is a
# MACHINE singleton (~/.dev-browser/daemon.sock) shared by every orchestrator — don't
# assume exclusive ownership; name browsers <sid>-<phase>, and give any dev-server a
# per-session port so concurrent runs don't clash.
dev-browser <<< 'console.log("daemon warm")'

cd "$INT"     # every later "main repo" step happens in the integration worktree
```
Start from a clean, committed state. Any foundation phase (Step 0) is committed
onto `int-$SID` here, before fan-out; runner diffs are reviewed against its HEAD.

### 2. Persistent worktrees (one per parallel phase)
```bash
# <preamble: SID/MAIN/INT — see Shell-state caveat>
git worktree add "$INT/.claude/worktrees/$SID-<phase>" -b wt-$SID-<phase> int-$SID
```
Each runner branches off `int-$SID`, inheriting any committed foundation phase.
The runner worktrees live *inside* `$INT` — harmless, because `.claude/worktrees/`
is excluded (Step 1), so `$INT`'s `git add -A` never swallows them as gitlinks.

**Warm the worktree with gitignored inputs.** A fresh worktree contains zero
gitignored files, so the runner has no `node_modules`, no `.env`, no generated
sources — the #1 cause of a runner wedging (`tsc: command not found`, missing
deps). Provide them *before launch*: the runner can't install them itself
(codex's seatbelt can't write the global package store). List what's
gitignored-but-present and classify it — the needed set is repo-specific, so
read it; don't assume just `node_modules`:
```bash
git ls-files -o -i --exclude-standard --directory   # gitignored & present
```
- **Clone the INPUTS** the toolchain reads: dependency dirs (`node_modules`,
  `.venv`, `vendor`, `Pods`), env/secret files (`.env*`, token-bearing
  `.npmrc`, service-account JSON), and gitignored-but-generated sources the
  build needs (codegen output, `expo-env.d.ts`, generated native projects).
- **Skip the OUTPUTS/ephemera**: `dist/`, `.next/`, build caches,
  `__pycache__`, logs, test artifacts, `.DS_Store` — and NEVER
  `.claude/worktrees/` (cloning it into a worktree recurses).

Clone copy-on-write so each worktree gets its own *writable* deps (the runner
can install a missing one without corrupting the source or sibling worktrees)
at near-zero disk on APFS/btrfs:
```bash
# <preamble: SID/MAIN/INT>. Source from $INT (the orchestrator's already-warmed tree,
# which carries any foundation-phase dep changes).
WT=$INT/.claude/worktrees/$SID-<phase>
warm() {  # warm <src-root> <relpath>
  local src=$1/$2 dst=$WT/$2
  [ -e "$src" ] || return 0
  mkdir -p "$(dirname "$dst")"
  cp -cR "$src" "$dst" 2>/dev/null \                   # macOS APFS clonefile (CoW)
    || cp -a --reflink=auto "$src" "$dst" 2>/dev/null \ # Linux btrfs/xfs reflink
    || ln -s "$src" "$dst"                             # last resort: shared read-only symlink
}
warm "$INT" node_modules; warm "$INT" surface/web/node_modules; warm "$INT" .env
```
`cp -c` / `--reflink=auto` self-fall-back to a *full* copy off-APFS or
cross-volume; for a huge dep dir on another volume, `ln -s` it instead
(shared, read-only — the runner then can't self-install into it).

### 3. Briefs
Write each brief to `/tmp/$SID-<phase>-brief.md` (avoids shell quoting issues;
`/tmp` is machine-global, so the `<sid>` prefix keeps concurrent runs from
clobbering each other's briefs, results, and patches). Template:

```
<one-paragraph goal and context>

You are working in <ABSOLUTE worktree path>. Run `pwd` first to confirm.
If this directory does not exist or is not a git worktree, STOP and report —
do not work anywhere else.

SCOPE — you own ONLY: <files / functions>. Additions to shared files go in an
appendix block marked `# === <phase> additions ===`. Touch nothing else.

Do NOT commit. The orchestrator reviews and merges your working-tree diff.

Your deps are a copy-on-write clone of the orchestrator's — installing a
missing one is fine and affects no one else. But if the toolchain is absent or
an install fails, do NOT loop on it: skip the test step, say so in your report,
and finish. The orchestrator re-verifies on the integration branch regardless.

TEST PROCEDURE (run before reporting):
<exact build/test commands>
# visual check (daemon is already running; browser name is globally unique):
cat > /tmp/<sid>-<phase>-shot.js <<'JS'
const page = await browser.getPage("<sid>-<phase>");
await page.setViewportSize({ width: 1440, height: 1000 });
await page.goto("file://<path under the worktree>");
await page.waitForTimeout(300);
console.log(await saveScreenshot(await page.screenshot({ fullPage: true }), "<sid>-<phase>.png"));
JS
dev-browser --browser <sid>-<phase> run /tmp/<sid>-<phase>-shot.js
# then VIEW the saved png and verify: <concrete visual criteria>

REPORT: what changed, how you tested (including what the screenshot showed),
known gaps.
```

### 4. Launch (one backgrounded Bash call per phase)
Each launch is its own Bash call, so start with the preamble. codex:
```bash
# <preamble: SID/MAIN/INT>
cd "$INT/.claude/worktrees/$SID-<phase>" && codex exec \
  --sandbox workspace-write \
  -c 'sandbox_workspace_write.network_access=true' \
  -c 'features.default_mode_request_user_input=false' \
  -o /tmp/$SID-<phase>-result.md \
  "$(cat /tmp/$SID-<phase>-brief.md)"
```
omp (e.g. Gemini Flash):
```bash
# <preamble: SID/MAIN/INT>
cd "$INT/.claude/worktrees/$SID-<phase>" && omp -p --no-session --auto-approve \
  --model gemini-3.5-flash \
  @/tmp/$SID-<phase>-brief.md > /tmp/$SID-<phase>-result.md
```
Run with `run_in_background: true`. The completion notification fires when the
process exits — do not write watcher loops.

### 5. Monitor (only when prompted or suspicious)
Tail the Bash output file. If no new output for ~10 minutes, check
`git -C <worktree> diff --stat` — runners sometimes wedge AFTER the work is
done. If the diff looks complete, kill the process and salvage the tree; the
work is rarely lost.

### 6. Review and merge (orchestrator, into `$INT` — never the primary checkout)
```bash
# <preamble: SID/MAIN/INT>
cd "$INT"
# stage in the runner tree first, then diff the INDEX — a plain `git diff` shows only
# tracked changes and would silently drop every NEW file the runner created. `add -A`
# can't pull in warmed deps (.env/node_modules are gitignored) or the nested worktrees
# (.claude/worktrees/ is excluded), so it captures exactly the runner's real work.
RWT=$INT/.claude/worktrees/$SID-<phase>
git -C "$RWT" add -A
git -C "$RWT" diff --cached > /tmp/$SID-<phase>.patch
# READ the patch, check the result file, then apply (—index so new files land staged):
git apply --3way --index /tmp/$SID-<phase>.patch
```
Re-run build/tests in `$INT` after each apply. Overlapping hunks across phases
mean the ownership split failed — resolve manually, don't re-delegate.

### 7. Final QA + integrate — yours, not the runners'
Runner screenshots are a first-line filter. From inside `$INT`, re-verify the
key surfaces yourself (desktop + mobile widths at minimum), then commit onto
`int-$SID` and land it — **PR if the repo has a remote, else local merge**:
```bash
# <preamble: SID/MAIN/INT>
cd "$INT"
git add -A && git commit -m "<summary>"

if git remote get-url origin >/dev/null 2>&1; then          # remote → PR (fully isolated)
  git push -u origin int-$SID
  gh pr create --fill --head int-$SID
else                                                         # no remote → local merge
  # lands on whatever branch $MAIN has checked out: confirm it's the target and clean,
  # else this fails or lands in the wrong place.
  git -C "$MAIN" diff --quiet && git -C "$MAIN" diff --cached --quiet \
    || echo "WARNING: primary checkout dirty — stash/commit there before merging"
  git -C "$MAIN" merge --ff-only int-$SID \
    || echo "ff-only refused: another run advanced main, or histories diverged — rebase int-$SID onto $MAIN's branch and retry"
fi

# cleanup: runner worktrees/branches first (they live inside $INT), then drop $INT.
# Use absolute paths — do NOT rely on cwd. Remove the worktree BEFORE its branch.
for p in <phases>; do
  git worktree remove "$INT/.claude/worktrees/$SID-$p" --force
  git branch -D wt-$SID-$p
done
cd "$MAIN"                                  # can't remove a worktree from inside it
git worktree remove "$INT" --force
# local merge → branch is now contained in main, safe-delete works; PR path → -d fails
# (not merged locally) and `|| true` keeps the branch alive for the open PR.
git branch -d int-$SID 2>/dev/null || true
```
The **PR path is fully isolated** — each run pushes its own `int-<sid>`; GitHub
serializes the merges. The **local-merge path is not**: `--ff-only` keeps history
linear and refuses (rather than tangling) when a concurrent run advanced main
first, but the landing still writes `$MAIN`'s one working tree, so concurrent
local landings serialize on its `index.lock` and a loser must rebase and retry.

## Failure-mode quick reference

| Symptom | Cause | Action |
|---|---|---|
| codex "runs" 10+ min with zero output AND zero diff (not an end-of-run wedge) | global config has `default_mode_request_user_input=true`; the headless lane is blocked asking a question no one can answer | launch with `-c 'features.default_mode_request_user_input=false'` (now in the Step-4 recipe); kill and salvage any partial diff |
| codex: "Daemon failed to start within 5 seconds" | daemon not pre-warmed, or `network_access` not set | warm daemon from orchestrator; relaunch with the `-c` flag |
| codex stuck retrying browser fallbacks (node_repl, NODE_PATH, --connect) | same as above | same; salvage any completed diff first |
| omp: "Use /login, set an API key environment variable..." | no stored credentials and no provider key in env | one-time `omp` + `/login`, or export the provider key; then relaunch |
| omp touched files outside its scope | no sandbox + loose brief | tighten SCOPE wording; review patch hunks before apply (you do this anyway) |
| runner wedges on `command not found` / missing `node_modules` | fresh worktree has no gitignored deps | warm the worktree (Step 2) before launch; brief skips-not-retries if deps absent |
| worktree vanished mid-task | it was harness-managed (Agent isolation) | kill orphans, recreate persistent worktree, relaunch |
| companion `status`: "No job found" | per-cwd state; checked from wrong dir | `cd` to the exact launch dir; never treat as completion |
| task "running" for an hour with frozen progress tail but full diff present | end-of-run wedge | kill, salvage tree, do verification yourself |
| stray gitlinks in a commit | `.claude/worktrees/` not excluded | pre-flight exclude line; fix commit with `git rm --cached` |
| `git worktree add` fails: "branch already exists" / "path already in use" | another orchestrator on the same repo grabbed that name | never use a bare phase name; `<sid>`-namespace every branch and path (Steps 1–2) |
| `merge --ff-only int-$SID` rejected | a concurrent orchestrator advanced main first | rebase `int-$SID` onto main and retry the merge (don't force) |
| two runs' screenshots/dev-servers clobber each other | shared daemon singleton + fixed port across orchestrators | `<sid>`-prefix browser names; assign a per-session dev-server port |
| one run's brief/patch overwrites another's in `/tmp` | `/tmp` is machine-global | `<sid>`-prefix every `/tmp` artifact (Steps 3–6) |
