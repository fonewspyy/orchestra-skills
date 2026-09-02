---
name: orchestra
description: Dispatch work to a local claude-orchestrator control plane, which runs Claude Code tasks across several logged-in accounts. Use when the user types /orchestra, asks to run something on another account, asks to parallelise or fan out work across accounts, mentions the orchestrator or the board, or has a job too large or too slow for one session. Also use to check what the pools are doing or to watch a dispatched task.
---

# orchestra

Dispatch a task to **claude-orchestrator**: a local control plane that runs Claude Code across
several logged-in accounts, with a queue, a live board, and a `PreToolUse` gate that holds a tool
call open until a human answers.

Everything below was read off a **running** server, not off its README. §9 records what was measured
where. Nothing here is hardcoded to the machine it was measured on — §0 resolves the handful of
values that differ, and the rest is the same everywhere.

## 0. Resolve this machine before anything else

Five things change per machine. Get them once at the start of the task and reuse them:

| | what | where it comes from |
|---|---|---|
| `$ORCH` | orchestrator install directory | §0.1 |
| `$PORT` | control-plane port | `4123` unless this machine's `.env` says otherwise |
| `$NODE` | the Node build whose ABI matches `better-sqlite3` | §0.2 |
| pools | account names, and which one the user is typing to | `GET /pools` — **never assume** |
| targets | repos in scope, how each verifies, what is production | `MACHINE.local.md` |

**`MACHINE.local.md` beside this file answers all five if it exists.** It is per-machine and
gitignored on purpose. If it is missing, resolve it below and then **write it** —
`MACHINE.example.md` in this folder is the template.

### 0.1 Find the orchestrator

In order, stopping at the first that answers:

1. `MACHINE.local.md` beside this SKILL.md. Trust it and skip the rest.
2. `$ORCHESTRA_HOME`, if this machine exports one.
3. Search. The signature is a directory whose `.env` defines `ORCHESTRATOR_TOKEN`:

```bash
for d in ~/claude-orchestrator ~/orchestrator ~/Desktop/*/ ~/src/*/ ~/projects/*/ ~/code/*/; do
  [ -f "$d/.env" ] && grep -qs '^ORCHESTRATOR_TOKEN=' "$d/.env" && echo "candidate: $d"
done
```

Found it by searching? **Write `MACHINE.local.md` before anything else**, so the next session on
this machine does not repeat the search. That file is the whole portability mechanism; skipping it
is what makes every future session re-derive the same five values.

Nothing found, and nothing answering on `$PORT`: this machine does not have the orchestrator. Say
so and stop. Do not install it uninvited.

### 0.2 The Node interpreter is not optional

`better-sqlite3` in the orchestrator tree is a **native** module compiled against one Node ABI. A
different major version refuses to load it:

```
NODE_MODULE_VERSION 141. This version of Node.js requires NODE_MODULE_VERSION 147
```

⚠ **That message means the interpreter is wrong, not that the tree is broken.** Do not `npm rebuild`
as a first move — find the Node that matches. `node -p "process.versions.modules"` prints the ABI of
any candidate, and the number in the error is the one you need.

⚠ On Windows a **user**-level `PATH` entry can never beat a **machine**-level one, so a system-wide
install wins over anything installed per-user, silently, every time. Export the right one explicitly
in every shell that touches the orchestrator:

```bash
export PATH="$NODE:$PATH"
```

Record the version that worked in `MACHINE.local.md` — that is what the field is for. The
orchestrator's own README and its RUNNING.md have contradicted each other about the required
version before. **The ABI error is the authority, not either document.**

## 1. Is it up?

```bash
curl -s -o /dev/null -w "%{http_code}" "http://127.0.0.1:$PORT/"
```

`200` means running. Anything else — start it:

```bash
export PATH="$NODE:$PATH"
cd "$ORCH" && npm run serve
```

Run it in the background. `npm run serve` does not return.

## 2. The API

Bearer auth. The token is `ORCHESTRATOR_TOKEN` in `$ORCH/.env`. **Read it, use it, never print it.**

```
GET  /pools                     every pool: account, health, 5-hour and 7-day utilization
POST /tasks                     { prompt, repoPath, pool? }  ->  201 { id, pool, crossedBoundary }
POST /tasks/:id/message         send more instruction to a running task — body is { text }, NOT { message }
POST /tasks/:id/stop            stop it
POST /tasks/:id/cancel          cancel a queued one
POST /tasks/:id/priority        reorder the queue
GET  /tasks                     every task: id, status, pool, cost_usd, duration_ms, error, …
GET  /threads                   conversations — threadId, active. NOT task state.
```

⚠ **`GET /tasks` is the only place task state lives.** `GET /tasks/:id` is a 404 — there is no
single-task route, so fetch the list and filter. `/threads` is keyed by `threadId`, has no `status`
field and does not contain a task id; a watcher pointed at it matches nothing forever and looks
exactly like a task that is still running.

Each task carries `status`, `cost_usd`, `duration_ms`, `error`, `failure_class`, `attempts` and
`crossed_boundary`. `cost_usd` is the number to quote when the user asks what a dispatch cost.

Only `prompt` and `repoPath` are required. Omit `pool` and the scope policy picks; pass it to
override.

PowerShell, which handles the token and quoting better than bash on Windows:

```powershell
$D = "<orchestrator dir>"
$tok = ((Get-Content "$D\.env" | Where-Object { $_ -match '^ORCHESTRATOR_TOKEN=' } | Select-Object -First 1) -replace '^ORCHESTRATOR_TOKEN=','').Trim('"',"'"," ")
$h = @{ Authorization = "Bearer $tok" }
$body = @{ prompt = $prompt; repoPath = "<target repo>"; pool = "<pool>" } | ConvertTo-Json -Depth 5
Invoke-WebRequest "http://127.0.0.1:<port>/tasks" -Method POST -Headers $h -Body $body -ContentType 'application/json' -UseBasicParsing
```

## 3. Choose the pool yourself — do not ask the user

⚠⚠ **FIRST, `GET /tasks` AND COUNT WHAT IS ALREADY RUNNING — INCLUDING OTHER SESSIONS' WORK.**
`/pools` tells you nothing about this. `utilization` is an **account quota**, not a **process
count**: a pool can read 0.06 five-hour and still have four agents live on the machine right now.

```bash
curl -s -H "Authorization: Bearer $TOK" "http://127.0.0.1:$PORT/tasks" \
 | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const r=JSON.parse(s).tasks.filter(t=>t.status==='running');console.log(r.length+' running');r.forEach(t=>console.log(' ',t.id.slice(0,8),t.pool,t.repo_path))})"
```

**Budget the MACHINE, not the accounts.** `maxConcurrent` is per *pool*, so with six pools the
control plane will happily run 30 Claude Code processes and nothing stops it. Keep the machine total
at **6 or fewer**, counting what is already there. At 6, wait — do not add "just a few more".

⚠ **This is written from an outage.** Another session was running 4 agents against a different repo;
I checked `/pools`, saw everything `ready` with low utilization, and dispatched 8 more. **Twelve
concurrent agents. The server process died** — not a task failure: all four survivors were left
`status='running'`, `attempts=1`, `error=NULL`, `failure_class=NULL`, with their last events inside
one 26-second window, and `permissions` had zero rows in two hours so it was not the §6a gate bug
either. Symptoms of the same class: `orchestrator.db` had reached **1.01 GB** over 512 tasks (~2 MB
of events each) and `better-sqlite3` is synchronous in-process, so twelve event streams hit one file.

⚠⚠ **THE CRASH KILLED THE OBSERVER, NOT THE WORK — and I got this wrong before checking.** I
reported four runs as lost and was about to re-dispatch them. Measured instead: every one was
**still alive**, writing `task_events` 0–10 seconds earlier. The `npm run serve` process dying does
**not** kill the Claude Code children it spawned; they keep running and keep writing to the
database. **Never re-dispatch a task because the control plane went away. Measure first.**
Re-dispatching would have doubled an already-overloaded machine.

⚠ **A restarted server does not reconcile, so `status` alone is worthless in both directions.**
Orphaned tasks stay `running` in the DB forever, and genuinely-live tasks also read `running`.
**The only honest liveness signal is `task_events` recency**, and it is what distinguishes the two:

```bash
export PATH="$NODE:$PATH"; cd "$ORCH"
node -e "const D=require('better-sqlite3');const db=new D('orchestrator.db',{readonly:true});const now=Date.now();
for(const t of db.prepare(\"SELECT id,pool,repo_path FROM tasks WHERE status='running'\").all()){
  const e=db.prepare('SELECT created_at FROM task_events WHERE task_id=? ORDER BY seq DESC LIMIT 1').get(t.id);
  const age=e?Math.round((now-e.created_at)/1000):null;
  console.log(t.id.slice(0,8),t.pool,age===null?'no events':age+'s',age!==null&&age<180?'LIVE':'orphan');}"
```

Under ~180s since the last event means live. Minutes of silence on a task that should be streaming
means orphaned — and *that* is when a re-dispatch is the right call, not before.

Then, among usable pools: pick the account with the **lowest `five_hour` utilization**, breaking
ties on `seven_day`, and skip any with `cooling: true` or `health` other than `ready`. Say which you
picked and why in one line.

⚠ **Never hardcode pool names — `GET /pools` is the list.** They are named per machine, and not all
of them are work pools; one is often scoped to a scratch directory only. Two extra checks:

- **Skip the account the user is currently typing to.** You compete with them for its quota, so it
  is usually the worst choice. `MACHINE.local.md` should name it.
- **Scope is per-machine and is in no repo.** Each pool has a list of paths it may touch, defined in
  `$ORCH/.env`, which is gitignored. A dispatch outside it returns **409** unless a pool is named
  explicitly, and then comes back `crossedBoundary: true`. So "which repos can I dispatch to" cannot
  be inferred from any checked-in file — read that `.env`, or try it and read the status code.

## 4. Writing the prompt is the whole job

The task runs with **no memory of this conversation**. A vague prompt wastes another account's
budget, which is the one cost the user actually feels.

Every dispatched prompt states:

- **The repo and the branch**, explicitly. Where several worktrees exist, which one is main and
  which are stale is not guessable from outside.
- **Which files it owns.** Sessions sharing one worktree share one staging area, and two have
  already buried each other's work. `git commit --only <paths>`, never `git add -A`, never
  `git reset --hard`.
- **How it verifies** — the exact command, and that it must come back green. `MACHINE.local.md`
  records the per-repo one.
- **What it must not do**: raise a ratchet ceiling, disable a guard, skip a test, widen a type to
  `any`, deploy, push, start a dev server, or touch a database. ⚠ Every machine has its own
  production hazard — a tunnel on some port, a live connection string. Name it in the prompt.
- **What to report back**, in numbers.
- **Which skill it loads.** Any dispatch that writes, refactors or reviews code carries `nohell` —
  see §4a. A read-only dispatch carries none.
- **The refutation clause** — see §6-bis. One sentence, and it has paid repeatedly.

Read the target repo's own work list first (`docs/TODO.md`, `docs/ROADMAP.md`, a status script —
whatever that repo keeps) and quote *its* current figures rather than a document's. Briefs written
from stale documents are the most common way a dispatch is wasted.

## 4a. Every code dispatch loads `nohell`

The dispatched process is Claude Code on this machine under this user, so it sees the same
`~/.claude/skills/` this session does, `nohell` included. Nothing in the spawn takes that away:
`src/runner/argv.ts` passes no skill or tool restriction, and the per-task `--settings` file from
`src/runner/settings.ts` writes hooks and nothing else.

Put this ahead of the task itself, as the first thing in the prompt:

```
Load the nohell skill at <absolute path to nohell/SKILL.md> and hold to it for this whole task:
walk the 7-step ladder before you write anything, and if you consolidate two things into one,
finish the C-cycle — the old copies get deleted, and net line count comes out negative. Report
the rule ids you hit and the ones you deliberately broke, with the reason.
```

**Resolve that absolute path on this machine first** — `~/.claude/skills/nohell/SKILL.md` is the
usual location; confirm the file exists, then paste the resolved path literally. Do not only say
"use nohell": a `-p` run has nobody to ask when a skill name fails to resolve, and it cannot tell
you it never loaded. Reading an absolute path either works or errors loudly.

⚠ **Do not attach it to a read-only dispatch.** `nohell/SKILL.md` is 16 KB and `HELL-CATALOG.md`
behind it is 187 KB. A task that only greps, counts or reports pays that and returns nothing for it.
Attach when the dispatch writes code, changes a stored procedure, or reviews someone else's work;
skip it when the dispatch only reads.

`ponytail` composes with it: `ponytail` asks whether the code needs to exist, `nohell` asks whether
it will become hell if it does. A dispatch that builds something new can name both — in that order,
which is the order `nohell`'s own ladder already assumes it ran in.

## 5. Watch it without burning tokens

Dispatch, then arm one Monitor. It polls, prints only on a state change, and each event is a line.
Run the inner `curl` once by hand before arming it.

```bash
ID=<task id from the 201>
TOK=$(grep '^ORCHESTRATOR_TOKEN=' "$ORCH/.env" | head -1 | cut -d= -f2- | tr -d '"'"'"' ')
prev=""
for i in $(seq 1 120); do
  cur=$(curl -s -m 10 -H "Authorization: Bearer $TOK" "http://127.0.0.1:$PORT/tasks" \
    | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const t=JSON.parse(s).tasks.find(x=>x.id===process.argv[1]);console.log(t?[t.status,t.cost_usd??0,t.duration_ms??0,t.error||''].join(' '):'NOT FOUND')})" "$ID")
  [ "$cur" != "$prev" ] && { echo "$ID $cur"; prev="$cur"; }
  case "$cur" in NOT*|done*|failed*|cancelled*|stopped*|error*) exit 0;; esac
  sleep 30
done
echo "watch window expired — task still $prev"
```

`NOT FOUND` is a terminal case on purpose: it means the id is wrong or the server restarted, and a
watcher that cannot see its task must say so rather than stay silent. Silence and "still running"
must never look the same — that is the failure this section exists to prevent.

`persistent: false`, `timeout_ms` matched to the job. It exits on a terminal state, so a finished
task ends the watch instead of leaving it armed.

**Never poll from the main loop.** A `sleep` in a Bash call blocks the turn and a repeated status
check costs a round trip each time; the Monitor costs one line per transition.

## 6. Fan out

For several independent streams, dispatch several tasks in one message — one per pool — and arm one
Monitor per task. The constraint is not the accounts, it is git: **sessions sharing one worktree
share one staging area**, and two have already buried each other's work. Give each its own.

⚠ **COUNT THE WORKTREES THAT ALREADY EXIST BEFORE CREATING ONE.** `git -C <repo> worktree list`.
On the machine this was written from there were **fourteen**, and an earlier version of this section
named three — so a session that trusts the document helpfully adds a fifteenth next to thirteen it
never looked at. Most sit idle at a stale commit on a branch named after whatever the run was
(`work/tax1`, `work/fifo1`, `work/zpark`) — **the branch name tells you nothing about what is in
it**, so check the tree, not the name.

Before reusing one, check BOTH, per worktree:

```bash
git -C <wt> status --porcelain | wc -l          # uncommitted work — someone's, maybe live
git -C <wt> rev-list --count main..HEAD         # commits not on main — unmerged work
```

`0` and `0` means free: `git -C <wt> reset --hard main` and dispatch into it. **Anything else, leave
it alone and pick another.** One worktree held 13 uncommitted files — a regenerated SDK from a run
the orchestrator's own crash had orphaned. Resetting it would have destroyed work nobody had a
record of, and its branch name said nothing about that.

⚠ **Never `git worktree remove --force` to clean up.** It follows the `node_modules` junction and
deletes the SHARED one in the main tree, breaking every live worktree at once. Delete the junction
first. Reuse beats removal — that is why there were fourteen.

⚠ **Reserve the main worktree's name.** Never name a worktree anything a careless glob or a
tab-complete can turn into the main checkout's path. Pick a naming scheme where main is not one
character away from a scratch tree.

Creating one — the shape that worked:

```bash
git -C <main> worktree add <wt> -b work/<name> main
cp <main>/<app>/.env <wt>/<app>/.env
```

```powershell
cmd /c mklink /J "<wt>\<app>\node_modules" "<main>\<app>\node_modules"
```

```bash
ln -s <main>/<app>/node_modules <wt>/<app>/node_modules   # macOS / Linux
```

A junction/symlink rather than a fresh install — the install is minutes per worktree and the trees
are identical. Confirm with the repo's own typecheck from inside the new worktree before dispatching
into it. Separate worktrees have separate index files, which is what makes concurrent commits safe.

Tell each task which worktree is its own **and** that the others exist, or one will helpfully "fix"
a file another is mid-way through. Nothing enforces this but the prompt.

## 6-quater. ⚠ FAN-OUT HAS A DISK COST, AND IT IS THE ONE NOBODY BUDGETS FOR

**Check free space before dispatching, and again after.** `node_modules` is junctioned and costs
nothing per worktree — but **the test-runner cache is not**, and every worktree that runs the gate
builds its own. Measured after twelve agents: the main tree 360 MB, and 13 more worktrees holding
44–144 MB each. Clearing the thirteen returned **1.21 GB**.

⚠ On that day the drive reached **4.8 MB free of 477 GB**, and the failure did not look like a full
disk. It looked like flaky tests:

```
ENOSPC: no space left on device, write        ← the only honest line, buried in one assertion
```

Everything else read as a timing problem — one suite failing 2-of-13 then 4-of-13 then passing,
taking 23–48 s instead of 7 s, a second suite joining it, ownership tests failing four assertions
and passing on an immediate rerun. Two sessions' worth of "the suite flakes under parallel load" was
partly this. **A full disk presents as nondeterminism, not as an error**, because only the tests that
happen to write a temp file fail, and which ones those are changes run to run.

So when several suites start flaking at once and the common factor looks like "load":

```bash
df -h .                                              # or Get-PSDrive on Windows
git -C <main> worktree list | awk '{print $1}' | while read w; do du -sm "$w"/**/.jest-cache 2>/dev/null; done
```

Test caches are regenerable by definition — deleting them is always safe and always reversible.
Delete the worktrees' first and keep the main tree's, since that is the one that runs the gate most.

⚠ Do NOT reach for `git worktree remove --force` to reclaim space. It follows the `node_modules`
junction and deletes the SHARED tree, breaking every live worktree at once.

## 6-ter. ⚠ NEVER PUT `git stash` IN A DISPATCHED PROMPT

**The stash stack is ONE per repository, not one per worktree.** It lives in `refs/stash` in the
common git dir, so every worktree pushes onto and pops off the *same* stack. Separate index files do
not save you here — that is the property that makes concurrent *commits* safe, and it does not
extend to the stash.

So `git stash push` … `git stash pop` — the obvious way to prove a test fails against the old code —
is a **cross-agent data race**. Nine agents doing it at once are popping each other's work off a
shared stack, LIFO, with no error and no warning.

⚠ This was discovered the way these things usually are: `git stash list` in the main tree showed
`WIP on work/u5` and `WIP on work/zpark` — two *other* worktrees' agents, plainly visible and
poppable. A push/pop pair happened to return the right stash only because nothing interleaved in
that second. And one agent had already **finished and left its stash behind**, so its work was
sitting orphaned on the shared stack.

The failing-first proof still matters. Tell agents to do it with a copy instead:

```bash
cp src/path/file.ts /tmp/keep.ts          # or the scratchpad
git checkout HEAD -- src/path/file.ts     # restore the pre-change version
<run the spec — it must fail>
cp /tmp/keep.ts src/path/file.ts          # put it back
git diff --stat -- src/path/file.ts       # prove the restore was exact
```

Worktree-local, no shared state, and the final `git diff --stat` proves nothing was lost. Put that
recipe in the prompt verbatim; agents reach for `git stash` by default if you don't.

## 6-bis. How wide to go — the ceiling is worktrees, not accounts

**Dispatch more tasks than there are pools.** The queue is real: excess tasks sit `queued` and start
as slots free. Nine concurrent were dispatched across six pools and the queue handled it without a
409. The number to size against is **free worktrees**, because two tasks in one worktree share a
staging area and will bury each other.

Volume is the owner's call, because it is their budget — and it has been relaxed before. What is
NOT relaxable, whatever the instruction, and must stay in every dispatched prompt:

- no production database writes — know what this machine's production hazard is before you write
  the prompt, and name it
- no raising a ratchet ceiling, no adding to an allowlist, no skipping a test, no `any`, no
  disabling a guard
- no push, no deploy, no dev server
- `git commit --only <paths>` — never `add -A`, never bare commit, never `reset --hard`

Those are safety, not convention. "Exceed the rules" means fan out wider, not ship unverified.

⚠ **Check the production hazard before writing the prompts, not after.** Where production is reached
through a tunnel, `(</dev/tcp/127.0.0.1/<port>)` says whether it is open. If it is CLOSED, every
task must be scoped CODE-ONLY, each prompt must say so and carry the figures you already measured.
Nine prompts that assume a live database against a closed tunnel is nine wasted dispatches, and each
one will spend its first minutes discovering the same thing.

**Put the refutation clause in every prompt.** "If the brief's premise turns out to be wrong, say so
with evidence and stop — a refuted premise is a good outcome." That clause paid three times in one
day: one agent proved a VAT diagnosis wrong and found the real mechanism, which the briefing session
had got backwards.

## 6a. The bug that killed every long dispatch — read this before "fixing" it again

Three sessions diagnosed this wrongly (twice as "trust", once as "a timeout") before anyone looked
at the database. **Diagnose from `permissions` and `task_events`, never from the error string.** The
string said `nothing on the stream`, which reads as a hang and was not one.

Two constants, each defensible alone, contradicted each other:

| file | value | what it did |
|---|---|---|
| `src/gate/permissions.ts` | **570s** | wait for a human to answer a permission card |
| `src/runner/input-limits.ts` | **300s + 30s** | shut a quiet pipe, then kill the child |

A run blocked on a card is quiet by definition, so the runner always shot first and the gate's
decision landed on a corpse. Five runs died at ~6 minutes, every one with its last stream event a
`tool_use` and a `permissions` row reading `status='denied', reason='no response'`, every one
retried 3–4× into the same wall. One was mid-refactor and left a half-applied edit that broke that
repo's typecheck for a day.

Fixed upstream: decision 45s, silence budget 15 min, and an unanswered card now **allows**
(`ORCHESTRATOR_GATE_UNATTENDED=deny` restores blocking; `unattended` on `createGate` overrides per
gate). The row is still written with a reason naming the policy — `--ungated` would record nothing.

⚠ **Check whether this machine's copy has the fix before assuming either behaviour.** The two
constants above are the thing to read; a checkout older than the fix still has the old pair.

To read the evidence yourself — and note the Node version or `better-sqlite3` will refuse:

```bash
export PATH="$NODE:$PATH"
cd "$ORCH"
# tables: tasks, task_events, permissions, deploy_runs, pool_settings, thread_meta
```

⚠ **`cost_usd: null` does not mean nothing happened.** Cost is recorded from the final `result`
event, so a killed run reads null however much work it did. One task wrote nine files and still
showed null. Judge progress from `task_events`, not cost.

⚠ **A healthy running task has `duration_ms` and `cost_usd` null too**, so a watcher that prints
only on change looks identical whether the run is working or wedged. Check `task_events` recency for
a heartbeat.

## 6b. Two ways this was broken in one day

⚠ **NEVER dispatch a task that edits the orchestrator's own running code.** The control plane was
started with `npm run serve`, then given a task to fix its classifier, retry policy and preflight.
The server exited part-way through and every running task was orphaned — the queue lost their
results, though the files they had already written survived in the target repo.

The task that succeeded that day touched only `.env` and markdown. The one that killed the server
touched `src/`. That is the whole difference.

If the orchestrator must fix itself: drain the queue first, stop the server, dispatch from a second
copy, or run the target under `serve:watch` on a different port. Never edit `src/` of a live server
that is executing your own task.

⚠ **A watcher must distinguish "no answer" from "the answer is: finished".** Mine reported
`all terminal` when the server had simply gone down: curl returned nothing, the JSON parse threw,
the status string came back empty, the grep for `running` found nothing, and empty was read as done.
An earlier version watched the wrong endpoint entirely and would have stayed silent forever.

So every watch loop needs three outcomes, not two — **progressing**, **finished**, and
**cannot tell**. Make unreachable loud:

```bash
raw=$(curl -s -m 10 -H "Authorization: Bearer $TOK" "http://127.0.0.1:$PORT/tasks") || raw=""
[ -z "$raw" ] && { echo "control plane unreachable — NOT a finished task"; exit 1; }
```

Silence and success must never render identically. That mistake was made three separate times in one
day, in three different scripts, each written by someone who had just documented the same trap
somewhere else.

## 7. When it misbehaves

Fix it at the source rather than teaching every caller to work around it — but **ask first if the
orchestrator checkout is not the user's own**. It is TypeScript with vitest, `pnpm test` and
`pnpm typecheck` at its root. Read its `HANDOFF.md` first; it carries the current state and the
ordering hazards. Two known-wrong things are recorded in §1 and §3 above.

## 8. Porting this skill to a new machine

1. Clone this repo and link `skills/orchestra` into `~/.claude/skills/` (see the repo README).
2. Copy `MACHINE.example.md` to `MACHINE.local.md` beside `SKILL.md` and fill it in. It is
   gitignored, so it never travels between machines — which is the point.
3. Fill it by measuring, not by copying another machine's: `GET /pools` for the pool names, the ABI
   error for the Node version, the repo's own scripts for the verify command.
4. If the machine has no orchestrator, stop there. The skill is a client; it does not install one.

⚠ **Do not carry another machine's `MACHINE.local.md` across.** Pool names, worktree layout and the
production hazard are exactly the values that are wrong-and-plausible on a different box — the class
of error that gets a dispatch pointed at the wrong repo or a prompt that omits a live database.

## 9. What was measured where

Everything above was measured on **one Windows machine, August 2026**, against one orchestrator
checkout, dispatching mostly at a Node/TypeScript monorepo. That is the provenance; treat it
accordingly:

| holds anywhere | holds only where measured |
|---|---|
| the API shape and its traps (`GET /tasks/:id` is 404, `/threads` is not task state, `{text}` not `{message}`) | the specific pool names, and which account is the owner's |
| liveness is `task_events` recency, never `status` | 12 agents being the number that killed the server — the ceiling is the machine, so measure yours |
| the crash kills the observer, not the children | 1.01 GB / 512 tasks — a size, not a limit |
| one stash stack per repository, across all worktrees | worktree names and how many exist |
| a full disk presents as flaky tests, not as an error | 1.21 GB reclaimed, 44–144 MB per cache |
| never dispatch a task that edits the running server's `src/` | whether this checkout still has the §6a timeout pair |
| a watcher needs three outcomes, not two | the verify command and the production hazard |

**When a number here disagrees with what you measure, the measurement wins.** Re-measure before
quoting any figure in this file to a user.
