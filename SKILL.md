---
name: agent-fanout
description: Delegate implementation work to parallel headless agent-CLI subagents (codex by default; omp with a cheap model like Gemini Flash as an alternative) in persistent git worktrees, with the current agent as orchestrator (plan, review, merge, final QA). Use when the user asks to delegate to codex or omp, fan out work to subagents, or parallelize implementation across cheaper agents. Includes the recipe for subagents doing their own visual QA via dev-browser.
---

# Agent fan-out delegation

The current agent is the orchestrator: decompose, write precise briefs, review
every diff, merge, and own final verification. Headless agent CLIs implement.
They are cheaper and faster, not smarter — never merge their work unreviewed.

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
- **codex**: launch through the bundled
  `scripts/launch-codex-lane` wrapper (Step 4), not raw `codex exec`. The
  wrapper always passes `--sandbox workspace-write` so global
  `~/.codex/config.toml` cannot accidentally run lanes unsandboxed, always
  passes `-c 'features.default_mode_request_user_input=false'` so headless
  clarification prompts do not block forever, runs Codex with `--json` so it
  can supervise child-bound progress, fails fast if Codex dead-starts or later
  goes quiet, and once progress is confirmed treats wrapper interruption as
  detach-not-kill, so a live lane survives orchestrator impatience. The final
  phase report still comes from `--result`; the JSON event log is diagnostic.
  Knobs:
  `--model <model>`, `--reasoning-effort high`.
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
   `<sid>`, creates `.worktrees/_int-<sid>` on branch `int-<sid>` from the
   freshly fetched remote target branch (`origin/HEAD` by default, or explicit
   `BASE_REF=origin/<branch>`), and does ALL "main repo" work there. Runner
   worktrees branch off `int-<sid>`; main is touched only by the final atomic
   merge/PR (Step 7). Every branch, worktree path, and `/tmp` artifact is
   `<sid>`-namespaced so concurrent runs never collide.
2. **Never combine the Agent tool's `isolation: worktree` with an async
   runner.** If the launching subagent exits before the runner finishes, the
   harness reaps the "unchanged" worktree and orphans the still-running task
   in a deleted directory. Create persistent worktrees manually (below).
3. **Launch runners directly via backgrounded Bash; completion = process
   exit.** No status-polling watcher loops. (History: codex-companion
   background jobs key their state per-cwd, so `status` from any other
   directory says "No job found" — which grep watchers misread as "finished".)
4. **A live runner is not failed just because it has no diff yet.** The Step-4
   wrapper owns liveness: it watches the child `codex exec --json` stream plus
   result-file and worktree changes, and exits `124` if the child never makes
   progress or later goes quiet for the configured timeout. Analysis,
   dependency inspection, and test discovery can still leave the worktree
   unchanged for several minutes, so do NOT stop a progress-confirmed runner
   for "no edits yet," and do NOT mark the phase completed from an empty diff.
   Only interrupt a started runner when the user asks you to stop or the runner
   is making destructive/out-of-scope changes. The wrapper enforces this
   boundary: after progress is confirmed, interrupting the wrapper
   (INT/TERM/HUP) detaches the lane — codex keeps running and the wrapper exits
   `130` printing the child pid. Stopping a confirmed lane is therefore always
   an explicit kill of that pid, allowed only for user stop,
   destructive/out-of-scope writes, or an adopted-runner quiet wedge after the
   wrapper has detached; a killed lane with no result file is abandoned, never
   completed. If you take a lane back with no diff, call it an abandoned lane,
   not a completed delegated phase, and either relaunch once with a tighter
   brief or explicitly report that you implemented it manually after abandoning
   the runner.
5. **Visual QA from inside a runner is the default — let runners self-QA;
   don't pull it back to the orchestrator to "avoid the sandbox wedge."** The
   wedge is fully prevented by ONE action you already do: pre-warm the
   dev-browser daemon from the orchestrator before launching (the Step-1
   `dev-browser <<< 'daemon warm'` line). The network half is automatic —
   `launch-codex-lane` passes `-c sandbox_workspace_write.network_access=true`
   by default (only `--no-network` disables it), so there is no extra flag to
   set by hand. With the daemon already up, codex lanes connect to
   `~/.dev-browser/daemon.sock` and screenshot normally; writes land in
   `~/.dev-browser/tmp/`, readable by all runners, so they can view their own
   PNGs and check their work.
   The wedge only happens if you SKIP the pre-warm: under codex's seatbelt the
   daemon cannot auto-start (Chrome dies at Mach-port registration) and the
   task may retry fallbacks for hours ("Daemon failed to start within 5
   seconds"). So the fix is to pre-warm, not to strip runner-side QA — that
   just discards a free first-line filter (the orchestrator still owns final
   QA in Step 7 regardless).

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
INT=$MAIN/.worktrees/_int-$SID
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
BASE_REF=${BASE_REF:-}            # optional override, e.g. BASE_REF=origin/release

# avoid the gitlink trap: embedded worktrees must never reach `git add -A`
# (.git/info/exclude lives in the shared common dir — once covers all worktrees)
grep -qx '.worktrees/' .git/info/exclude 2>/dev/null \
  || echo '.worktrees/' >> .git/info/exclude

# choose the remote target, not the possibly stale/dirty local checkout.
# If this repo has no origin, fall back to local HEAD for offline/local-only work.
if git remote get-url origin >/dev/null 2>&1; then
  git fetch --prune origin
  if [ -z "$BASE_REF" ]; then
    BASE_REF=$(git ls-remote --symref origin HEAD \
      | awk '$1 == "ref:" {sub("^refs/heads/","",$2); print "origin/" $2; exit}')
  fi
  if [ -z "$BASE_REF" ]; then
    BASE_REF=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null || true)
  fi
  [ -n "$BASE_REF" ] || { echo "No origin default branch; set BASE_REF=origin/<branch>"; exit 1; }
  case "$BASE_REF" in
    origin/*) git fetch origin "${BASE_REF#origin/}:refs/remotes/$BASE_REF" ;;
  esac
else
  test -z "$(git status --porcelain --untracked-files=normal)" \
    || { echo "Primary checkout dirty and no origin exists; commit/stash first"; exit 1; }
  BASE_REF=HEAD
fi
git rev-parse --verify "$BASE_REF^{commit}" >/dev/null \
  || { echo "Base ref not found: $BASE_REF"; exit 1; }

# the orchestrator's OWN isolated checkout; the primary checkout is never mutated
# until Step 7. Other orchestrators get their own _int-<their-sid> — no contention.
git worktree add .worktrees/_int-$SID -b int-$SID "$BASE_REF"
INT=$MAIN/.worktrees/_int-$SID
git -C "$INT" config "branch.int-$SID.agentFanoutBase" "$BASE_REF"

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
# and any compiler caches a phase will need — discover them; see Step 2 for why
# these are worth cloning and how to enumerate them for your toolchain.

# if any phase needs visual QA: warm the daemon OUTSIDE any sandbox. The daemon is a
# MACHINE singleton (~/.dev-browser/daemon.sock) shared by every orchestrator — don't
# assume exclusive ownership; name browsers <sid>-<phase>, and give any dev-server a
# per-session port so concurrent runs don't clash.
dev-browser <<< 'console.log("daemon warm")'

cd "$INT"     # every later "main repo" step happens in the integration worktree
```
Start from a clean, committed remote target. With `origin`, the integration
branch deliberately starts from `origin/HEAD` (or your explicit
`BASE_REF=origin/<branch>`), not from the local checkout's current `HEAD`. Any
foundation phase (Step 0) is committed onto `int-$SID` here, before fan-out;
runner diffs are reviewed against its HEAD.

### 2. Persistent worktrees (one per parallel phase)
```bash
# <preamble: SID/MAIN/INT — see Shell-state caveat>
git worktree add "$INT/.worktrees/$SID-<phase>" -b wt-$SID-<phase> int-$SID
```
Each runner branches off `int-$SID`, inheriting any committed foundation phase.
The runner worktrees live *inside* `$INT` — harmless, because `.worktrees/`
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
- **Skip the OUTPUTS/ephemera**: `dist/`, `.next/`, `__pycache__`, logs, test
  artifacts, `.DS_Store` — and NEVER `.worktrees/` (cloning it into a worktree
  recurses).
- **But DO clone the expensive COMPILER caches** — cargo's `target/`, and
  anything like it. This is the one "build cache" you must not skip. A fresh
  worktree has an empty `target/`, so every Rust lane recompiles the entire
  dependency graph before it runs a single test, and N parallel lanes do it N
  times while fighting each other for CPU. It is usually the largest single
  source of dead wall-clock in a fanout. Measured on a 528-crate workspace:

  | `cargo test --no-run` in a fresh worktree | wall-clock |
  |---|---|
  | empty `target/` (what you get if you skip this) | **1:42** |
  | `target/` cloned from the main checkout (clone itself: 2.3s) | **0:12** |

  Cloning is safe because cargo fingerprints every unit and revalidates on each
  build — a stale or mismatched artifact gets rebuilt, never silently reused.
  Copy-on-write makes it near-free: a 3 GB `target/` clones in ~2s and consumes
  ~0 extra disk until something rewrites a block.

  Generalize by *property, not by name*: clone a cache when it is *expensive to
  regenerate* AND *self-validating* (the tool checks freshness itself). Cargo
  `target/`, Gradle's caches, and a built `.venv` qualify. `dist/`, `.next/`,
  and friends do not — cheap to rebuild, and not revalidated. Keep skipping those.

  **Do not reach for `sccache`/`RUSTC_WRAPPER` to solve this.** Its cache key
  includes the target-dir path, so worktrees at different absolute paths share
  *zero* hits (measured: same path wiped = 100% hit; different path = 0%). Worse,
  setting `RUSTC_WRAPPER` changes every unit's fingerprint, so turning it on
  *invalidates* a cloned `target/` and forces the full rebuild you were avoiding.
  Clone the directory; leave the wrapper alone.

Clone copy-on-write so each worktree gets its own *writable* deps (the runner
can install a missing one without corrupting the source or sibling worktrees)
at near-zero disk on APFS/btrfs:
```bash
# <preamble: SID/MAIN/INT>. Source from $INT (the orchestrator's already-warmed tree,
# which carries any foundation-phase dep changes).
WT=$INT/.worktrees/$SID-<phase>
warm() {  # warm <src-root> <relpath>
  local src=$1/$2 dst=$WT/$2
  [ -e "$src" ] || return 0
  mkdir -p "$(dirname "$dst")"
  # Try APFS clonefile, then Linux reflink, then a shared symlink fallback.
  cp -cR "$src" "$dst" 2>/dev/null \
    || cp -a --reflink=auto "$src" "$dst" 2>/dev/null \
    || ln -s "$src" "$dst"
}
warm "$INT" node_modules; warm "$INT" surface/web/node_modules; warm "$INT" .env

# Compiler caches — discover them, don't hardcode. A cache dir is only worth
# cloning if it already exists in the source tree (i.e. that toolchain has been
# built at least once). Cargo example; the same shape works for any ecosystem:
while read -r d; do warm "$MAIN" "$d"; done < <(
  cd "$MAIN" && git ls-files '*Cargo.toml' | xargs -n1 dirname | sort -u \
    | while read -r c; do [ -d "$c/target" ] && echo "$c/target"; done
)
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

Do not spin indefinitely in analysis. Once you understand the local pattern,
make the scoped change, run the requested checks, and report. If you are
blocked or believe no code change is needed, write that conclusion to the
result and exit instead of continuing to inspect files.

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
Each launch is its own Bash call, so start with the preamble. Use the bundled
Codex launch wrapper from this skill directory; set `SKILL_DIR` to the base
directory printed when this skill is loaded. The wrapper supervises the child
`codex exec`: it runs Codex with `--json`, writes the final phase report to
`--result`, and watches child-bound progress from JSONL events, result-file
creation, and worktree changes. If no progress appears within
`--startup-timeout`, or if progress later stops for `--quiet-timeout`, it kills
the process group, writes a diagnostic result file, and exits `124`. Rollout
files are used only as child-bound diagnostics (`session_meta.cwd` must equal
the lane worktree and `originator` must be `codex_exec`), never as a parent
transcript substring match. Signals flip meaning at progress confirmation:
before it, interrupting the wrapper kills the child (a dead start is not worth
preserving); after it, the wrapper detaches instead — it prints the child pid
and file paths, exits `130`, and codex keeps running.

codex:
```bash
# <preamble: SID/MAIN/INT>
SKILL_DIR=/path/to/agent-fanout   # use the base directory printed when the skill loaded
WT=$INT/.worktrees/$SID-<phase>
"$SKILL_DIR/scripts/launch-codex-lane" \
  --worktree "$WT" \
  --brief /tmp/$SID-<phase>-brief.md \
  --result /tmp/$SID-<phase>-result.md \
  --run-log /tmp/$SID-<phase>-run.log \
  --startup-timeout 600 \
  --quiet-timeout 600
```
omp (e.g. Gemini Flash):
```bash
# <preamble: SID/MAIN/INT>
cd "$INT/.worktrees/$SID-<phase>" && omp -p --no-session --auto-approve \
  --model gemini-3.5-flash \
  @/tmp/$SID-<phase>-brief.md > /tmp/$SID-<phase>-result.md
```
Run with `run_in_background: true`. The completion notification fires when the
wrapper exits — which is NOT always lane completion: exit `130` after
"progress confirmed" means the wrapper detached and the runner is still working
(Step 5). Do not write watcher loops, and do not interrupt a
progress-confirmed runner just because repeated diff checks show no edits. Empty
diff while the process is still running means "not done yet," not "failed."

### 5. Monitor (only when prompted, or when a real failure signal exists)
Tail the Bash output file, `/tmp/$SID-<phase>-run.log`, and the adjacent
`/tmp/$SID-<phase>-run.log.events.jsonl` only when you need diagnostics. The
Codex wrapper catches dead-start and quiet wedges automatically: exit `124`
means Codex was alive but failed the child-progress watchdog. Read the
diagnostic result file and either relaunch once or take the lane back manually.

Use this decision rule:
- **Process exited:** review its result file and staged patch in Step 6.
- **Exit `124`:** the child never produced progress or later went quiet; read
  diagnostics, then relaunch once or take the lane back manually.
- **Exit `130` after "progress confirmed":** the wrapper was interrupted and
  detached; the runner is STILL RUNNING. Adopt it —
  `while kill -0 <codex-pid> 2>/dev/null; do sleep 20; done` — then read the
  result file as usual. Do not relaunch, and do not treat this as a failed
  lane.
- **Still running under the wrapper:** keep waiting. The wrapper owns quiet
  detection; empty diff while alive is normal during analysis.
- **Adopted/detached runner quiet for 10+ minutes:** check the diff and result.
  If the diff looks complete, kill the process and salvage the tree; runners
  sometimes wedge after finishing the work. If there is still no diff, do not
  call the phase complete. Relaunch once with a tighter brief, ask the user, or
  take the lane back manually and report that the runner was abandoned.
- **Out-of-scope or destructive writes:** stop that runner immediately, discard
  its patch, tighten the brief, and relaunch or implement manually.

### 6. Review and merge (orchestrator, into `$INT` — never the primary checkout)
```bash
# <preamble: SID/MAIN/INT>
cd "$INT"
# stage in the runner tree first, then diff the INDEX — a plain `git diff` shows only
# tracked changes and would silently drop every NEW file the runner created. `add -A`
# can't pull in warmed deps (.env/node_modules are gitignored) or the nested worktrees
# (.worktrees/ is excluded), so it captures exactly the runner's real work.
RWT=$INT/.worktrees/$SID-<phase>
git -C "$RWT" add -A
git -C "$RWT" diff --cached > /tmp/$SID-<phase>.patch
# READ the patch, check the result file, then apply (—index so new files land staged):
git apply --3way --index /tmp/$SID-<phase>.patch
```
An empty patch is not automatically success. Treat it as success only if the
runner's result file explicitly explains why no code change was needed and you
agree after inspection. Otherwise relaunch once with a tighter brief or take the
lane back manually, and report that no runner diff was merged. Re-run
build/tests in `$INT` after each apply. Overlapping hunks across phases mean
the ownership split failed — resolve manually, don't re-delegate.

### 7. Final QA + integrate — yours, not the runners'
Runner screenshots are a first-line filter. From inside `$INT`, run the
orchestrator's own gate BEFORE landing — it's yours because it crosses the
phase boundaries no single runner owns:
- Re-verify the key surfaces yourself (desktop + mobile widths at minimum).
- Drive the suite to green: build, unit/integration, **and the repo's e2e
  suite** (Playwright/Cypress/etc.). Locate it before assuming it's absent —
  grep `package.json` scripts (`test:e2e`, `e2e`, `playwright`, `cypress`) or
  the CI workflow; if there genuinely is none, say so and fall back to
  build/tests + visual. On the **local-merge path there is no CI**, so this
  local run is the ONLY gate and it MUST include e2e. A red suite blocks the
  land: fix it or drop the offending phase — never merge around it.

Then commit onto `int-$SID` and land it — **PR (land on green CI) if the repo
has a remote, else local merge**:
```bash
# <preamble: SID/MAIN/INT>
cd "$INT"
git add -A && git commit -m "<summary>"

if git remote get-url origin >/dev/null 2>&1; then          # remote → PR, land on GREEN CI
  BASE_REF=$(git config --get branch.int-$SID.agentFanoutBase || true)
  BASE_BRANCH=${BASE_REF#origin/}
  [ -n "$BASE_REF" ] && [ "$BASE_BRANCH" != "$BASE_REF" ] \
    || { echo "Missing origin base ref for int-$SID; inspect branch.int-$SID.agentFanoutBase"; exit 1; }
  git push -u origin int-$SID
  gh pr create --fill --base "$BASE_BRANCH" --head int-$SID
  # land green: block on required checks, THEN merge. Never merge a red/pending PR.
  # --watch polls the checks to completion; --fail-fast bails on the first failing one.
  if gh pr checks int-$SID --watch --fail-fast; then
    gh pr merge int-$SID --squash --delete-branch          # respects branch protection; drops local+remote int-$SID
  else
    echo "CI not green — inspect \`gh pr checks int-$SID\`; leave the PR open, do NOT merge"
  fi
  # (repos with auto-merge enabled: \`gh pr merge --squash --auto --delete-branch\` also works)
else                                                         # no remote → local merge (your local gate above IS the CI)
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
  git worktree remove "$INT/.worktrees/$SID-$p" --force
  git branch -D wt-$SID-$p
done
cd "$MAIN"                                  # can't remove a worktree from inside it
git worktree remove "$INT" --force
# PR path: --delete-branch already dropped local+remote int-$SID (squash-merged, so a
# plain `-d` would refuse). local-merge path: branch is contained in main and `-d` works.
git branch -d int-$SID 2>/dev/null || true

# stale sweep — reclaim leftovers from earlier crashed/aborted runs WITHOUT touching a
# live concurrent orchestrator. Safe because each command only removes what git proves dead:
git -C "$MAIN" worktree prune                              # clears admin entries for worktrees whose dir is already gone
git -C "$MAIN" fetch -p origin 2>/dev/null || true         # marks branches whose remote was deleted as [gone]
git -C "$MAIN" for-each-ref --format '%(refname:short) %(upstream:track)' refs/heads \
  | awk '$2=="[gone]"{print $1}' \
  | xargs -r -n1 git -C "$MAIN" branch -D                  # delete ONLY [gone] branches (merged PRs) — never an in-flight run's
# Local-only orphans (wt-*/int-* from a run that crashed before pushing) can't be told apart
# from a live run by name, so they are NOT auto-swept — clear those by hand, or via the
# dedicated clean_gone skill, once you've confirmed no run still owns them.
```
The **PR path is fully isolated** — each run pushes its own `int-<sid>`; GitHub
serializes the merges, and required-checks CI is the authoritative green gate
(the local suite above is a pre-filter). The **local-merge path is not** isolated
and has **no CI** — your local build/tests/e2e is the whole gate: `--ff-only`
keeps history linear and refuses (rather than tangling) when a concurrent run
advanced main first, but the landing still writes `$MAIN`'s one working tree, so
concurrent local landings serialize on its `index.lock` and a loser must rebase
and retry.

## Failure-mode quick reference

| Symptom | Cause | Action |
|---|---|---|
| codex wrapper exits `124` before first progress | Codex launched and stayed alive, but produced no child-bound progress before `--startup-timeout`: no useful `--json` event, no result file, no worktree diff | read `/tmp/$SID-<phase>-result.md`, `/tmp/$SID-<phase>-run.log`, and `/tmp/$SID-<phase>-run.log.events.jsonl`; relaunch once or take the lane back manually |
| codex wrapper exits `124` after progress | Codex produced progress, then no child-bound progress for `--quiet-timeout` | inspect the diagnostic result and patch; salvage only if the diff/result are complete, otherwise relaunch once or take the lane back manually |
| parent/orchestrator transcript contains the lane worktree path | expected: launch commands mention the worktree, but parent transcript growth is not child progress | wrapper ignores parent substring matches; child rollout diagnostics require `session_meta.cwd == worktree` and `originator == codex_exec` |
| raw codex "runs" 10+ min with zero output AND zero diff | launched without the Step-4 wrapper, or the wrapper detached and is no longer supervising | kill it; relaunch through `scripts/launch-codex-lane` |
| progress-confirmed runner is still running with zero diff | normal analysis/setup period, not a failed lane | keep waiting while the wrapper is supervising; only interrupt for user stop or out-of-scope/destructive writes |
| wrapper exited `130` printing "interrupted after progress; codex is still running" | wrapper got INT/TERM/HUP after progress confirmation — it detaches by design instead of killing the lane | adopt the live runner: `while kill -0 <codex-pid>; do sleep 20; done`, then read the `--result` file; kill the pid only for user stop, destructive/out-of-scope writes, or an adopted-runner quiet wedge |
| runner was interrupted with zero diff | orchestrator abandoned the lane before it completed | do not mark it delegated/completed; relaunch once with a tighter brief or state that the implementation was manual |
| codex: "Daemon failed to start within 5 seconds" | daemon not pre-warmed before launch (network is on by default via the wrapper, so it's almost always the missing pre-warm — unless you passed `--no-network`) | pre-warm the daemon from the orchestrator (Step 1), then relaunch; don't strip runner-side QA to dodge it |
| codex stuck retrying browser fallbacks (node_repl, NODE_PATH, --connect) | same as above | same; salvage any completed diff first |
| omp: "Use /login, set an API key environment variable..." | no stored credentials and no provider key in env | one-time `omp` + `/login`, or export the provider key; then relaunch |
| omp touched files outside its scope | no sandbox + loose brief | tighten SCOPE wording; review patch hunks before apply (you do this anyway) |
| runner wedges on `command not found` / missing `node_modules` | fresh worktree has no gitignored deps | warm the worktree (Step 2) before launch; brief skips-not-retries if deps absent |
| wrapper dies with a bash syntax error mid-run while codex keeps working | the launcher script was EDITED while an instance ran (bash reads incrementally; fixed by the self-copy re-exec — affects only pre-fix instances) | adopt the orphan: `while kill -0 <codex-pid>; do sleep 20; done` then read the `--result` file as usual |
| worktree vanished mid-task | it was harness-managed (Agent isolation) | kill orphans, recreate persistent worktree, relaunch |
| companion `status`: "No job found" | per-cwd state; checked from wrong dir | `cd` to the exact launch dir; never treat as completion |
| task "running" for an hour with frozen progress tail but full diff present | end-of-run wedge | kill, salvage tree, do verification yourself |
| stray gitlinks in a commit | `.worktrees/` not excluded | pre-flight exclude line; fix commit with `git rm --cached` |
| `git worktree add` fails: "branch already exists" / "path already in use" | another orchestrator on the same repo grabbed that name | never use a bare phase name; `<sid>`-namespace every branch and path (Steps 1–2) |
| `merge --ff-only int-$SID` rejected | a concurrent orchestrator advanced main first | rebase `int-$SID` onto main and retry the merge (don't force) |
| `gh pr checks --watch` exits non-zero / PR stuck pending | a required CI check failed or never started | inspect `gh pr checks int-$SID`; fix the failure on `int-$SID` and push, or leave the PR open — never merge past a red gate |
| e2e "passed" but the suite never actually ran | assumed absent without looking | grep `package.json`/CI for `test:e2e`/`playwright`/`cypress` before landing; only fall back to build+visual if there's genuinely none |
| stale sweep left a dead `wt-*`/`int-*` behind | local-only orphan from a crashed run — indistinguishable from a live run by name | confirm no run owns it, then remove by hand; `worktree prune` + the `[gone]` delete only touch git-proven-dead refs |
| two runs' screenshots/dev-servers clobber each other | shared daemon singleton + fixed port across orchestrators | `<sid>`-prefix browser names; assign a per-session dev-server port |
| one run's brief/patch overwrites another's in `/tmp` | `/tmp` is machine-global | `<sid>`-prefix every `/tmp` artifact (Steps 3–6) |
