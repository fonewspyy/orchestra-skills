---
name: orchestra
description: Dispatch work to the local claude-orchestrator control plane, which runs Claude Code tasks across six logged-in accounts. Use when the user types /orchestra, asks to run something on another account, asks to parallelise or fan out work across accounts, mentions the orchestrator or the board, or has a job too large or too slow for one session. Also use to check what the pools are doing or to watch a dispatched task.
---

# orchestra

Dispatch a task to `claude-orchestrator` at `C:\Users\Wassaphas\Desktop\cluade-orchestra`. It runs
Claude Code across six accounts with a queue, a live board on `http://127.0.0.1:4123`, and a
`PreToolUse` gate that holds a tool call open until a human answers.

**Verified 2026-08-08 by dispatching a real task and getting a 201.** Everything below was read off
the running server, not from its README.

## 1. Is it up?

```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:4123/
```

`200` means running. Anything else — start it, and note the Node requirement, which is not optional:

```bash
export PATH="/c/Users/Wassaphas/nodejs/node-v26.7.0-win-x64:$PATH"
cd /c/Users/Wassaphas/Desktop/cluade-orchestra && npm run serve
```

⚠ **Node v26.7.0 only.** `better-sqlite3` in that tree is an ABI 147 build. The machine `PATH`
resolves `node` to v25.1.0 from `C:\Program Files\nodejs`, which is end-of-life and ABI 141, and a
user-level `PATH` entry can never win against a machine entry on Windows — so the export above is
required every time. `NODE_MODULE_VERSION … requires <n>` means the interpreter is wrong, not the
tree. *(That repo's README says Node 22 and its RUNNING.md says Node 26. RUNNING.md is right.)*

Run it in the background — `npm run serve` does not return.

## 2. The API

Bearer auth. The token is `ORCHESTRATOR_TOKEN` in that repo's `.env`. **Read it, use it, never
print it.**

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
exactly like a task that is still running. That mistake was made and corrected on 2026-08-08.

Each task carries `status`, `cost_usd`, `duration_ms`, `error`, `failure_class`, `attempts` and
`crossed_boundary`. `cost_usd` is the number to quote when the user asks what a dispatch cost.

Only `prompt` and `repoPath` are required. Omit `pool` and the scope policy picks; pass it to
override.

PowerShell, which handles the token and quoting better than bash here:

```powershell
$D = "C:\Users\Wassaphas\Desktop\cluade-orchestra"
$tok = ((Get-Content "$D\.env" | Where-Object { $_ -match '^ORCHESTRATOR_TOKEN=' } | Select-Object -First 1) -replace '^ORCHESTRATOR_TOKEN=','').Trim('"',"'"," ")
$h = @{ Authorization = "Bearer $tok" }
$body = @{ prompt = $prompt; repoPath = "C:/mm-t"; pool = "account_d" } | ConvertTo-Json -Depth 5
Invoke-WebRequest "http://127.0.0.1:4123/tasks" -Method POST -Headers $h -Body $body -ContentType 'application/json' -UseBasicParsing
```

## 3. Choose the pool yourself — do not ask the user

⚠⚠ **FIRST, `GET /tasks` AND COUNT WHAT IS ALREADY RUNNING — INCLUDING OTHER SESSIONS' WORK.**
`/pools` tells you nothing about this. `utilization` is an **account quota**, not a **process
count**: a pool can read 0.06 five-hour and still have four agents live on the machine right now.

```bash
curl -s -H "Authorization: Bearer $TOK" http://127.0.0.1:4123/tasks \
 | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const r=JSON.parse(s).tasks.filter(t=>t.status==='running');console.log(r.length+' running');r.forEach(t=>console.log(' ',t.id.slice(0,8),t.pool,t.repo_path))})"
```

**Budget the MACHINE, not the accounts.** `maxConcurrent` is 5 *per pool* and there are six pools,
so the control plane will happily run 30 Claude Code processes and nothing stops it. Keep the
machine total at **6 or fewer**, counting what is already there. If the count is already at 6, wait
— do not add "just a few more".

⚠ **This is written from an outage on 2026-08-11.** Another session was running 4 agents against a
different repo (`D:/PHONEEEE…/EWM_RF_ConfirmPack`); I checked `/pools`, saw everything `ready` with
low utilization, and dispatched 8 more. **Twelve concurrent agents. The server process died** — not
a task failure: all four survivors were left `status='running'`, `attempts=1`, `error=NULL`,
`failure_class=NULL`, with their last events inside one 26-second window, and `permissions` had zero
rows in two hours so it was not the §6a gate bug either. Symptoms of the
same class: `orchestrator.db` had reached **1.01 GB** over 512 tasks (~2 MB of events each) and
`better-sqlite3` is synchronous in-process, so twelve event streams hit one file.

⚠⚠ **THE CRASH KILLED THE OBSERVER, NOT THE WORK — and I got this wrong before checking.** I
reported four reports as lost and was about to re-dispatch them. Measured instead: every one of the
five was **still alive**, writing `task_events` 0–10 seconds earlier. The `npm run serve` process
dying does **not** kill the Claude Code children it spawned; they keep running and keep writing to
the database. **Never re-dispatch a task because the control plane went away. Measure first.**
Re-dispatching would have doubled an already-overloaded machine.

⚠ **A restarted server does not reconcile, so `status` alone is worthless in both directions.**
Orphaned tasks stay `running` in the DB forever, and genuinely-live tasks also read `running`.
**The only honest liveness signal is `task_events` recency**, and it is what distinguishes the two:

```bash
node -e "const D=require('better-sqlite3');const db=new D('orchestrator.db',{readonly:true});const now=Date.now();
for(const t of db.prepare(\"SELECT id,pool,repo_path FROM tasks WHERE status='running'\").all()){
  const e=db.prepare('SELECT created_at FROM task_events WHERE task_id=? ORDER BY seq DESC LIMIT 1').get(t.id);
  const age=e?Math.round((now-e.created_at)/1000):null;
  console.log(t.id.slice(0,8),t.pool,age===null?'no events':age+'s',age!==null&&age<180?'LIVE':'orphan');}"
```

Under ~180s since the last event means live. Minutes of silence on a task that should be streaming
means orphaned — and *that* is when a re-dispatch is the right call, not before.

Then, among pools that are usable: pick the account with the **lowest `five_hour`
utilization**, breaking ties on `seven_day`, and skip any with `cooling: true` or `health` other
than `ready`. Say which you picked and why in one line.

The six as configured: `team1` (claude.tsv — **this is usually the account the user is typing to,
so it is usually the worst choice**), `team2`, `account_c`, `account_d`, `account_e`, and `scratch`
(scoped only to `~/scratch`, not a work pool).

Every pool's scope now covers `C:/mm-t/**` and `C:/mm-a..d/**` as well as the Desktop folder —
fixed 2026-08-09 in that repo's `.env`, which is gitignored, so it lives only on this machine
(`.env.bak-20260809` is the copy from before). Before that, dispatching to `C:/mm-t` returned 409
unless a pool was named explicitly and came back `crossedBoundary: true`.

## 4. Writing the prompt is the whole job

The task runs with no memory of this conversation. A vague prompt wastes another account's budget,
which is the one cost the user actually feels.

Every dispatched prompt states:

- **The repo and branch**, and that `C:/mm-t` is `main` while the Desktop folder is a stale branch.
- **Which files it owns.** Sessions share one worktree and one staging area; two have already
  buried each other's work here. `git commit --only <paths>`, never `git add -A`, never
  `git reset --hard`.
- **How it verifies** — usually `npm run gate` from `C:/mm-t/server`, and it must be green.
- **What it must not do**: raise a ratchet ceiling, disable a guard, skip a test, widen a type to
  `any`, deploy, push, start a dev server, or touch a database. ⚠ Port 3307 is an SSH tunnel to
  the shop's **production** database.
- **What to report back**, in numbers.
- **Which skill it loads.** Any dispatch that writes, refactors or reviews code carries `nohell` —
  see §4a. A read-only dispatch carries none.

Read `C:/mm-t/docs/TODO.md` and `C:/mm-t/docs/ROADMAP.md` before writing one; they hold the work
list and the reason for its order. `node C:/mm-t/scripts/ssot-status.mjs` prints every current
figure — quote it rather than a document.

## 4a. Every code dispatch loads `nohell`

The dispatched process is Claude Code on this machine under this user, so it sees
`C:/Users/Wassaphas/.claude/skills/` exactly as this session does, `nohell` included. Nothing in
the spawn takes that away: `src/runner/argv.ts` passes no skill or tool restriction, and the
per-task `--settings` file from `src/runner/settings.ts` writes hooks and nothing else.
**Verified 2026-08-26 by reading both files.**

Put this ahead of the task itself, as the first thing in the prompt:

```
Load the nohell skill at C:/Users/Wassaphas/.claude/skills/nohell/SKILL.md and hold to it for
this whole task: walk the 7-step ladder before you write anything, and if you consolidate two
things into one, finish the C-cycle — the old copies get deleted, and net line count comes out
negative. Report the rule ids you hit and the ones you deliberately broke, with the reason.
```

Name the path; do not only say "use nohell". A `-p` run has nobody to ask when a skill name fails
to resolve, and it cannot tell you it never loaded — reading an absolute path either works or
errors loudly.

⚠ **Do not attach it to a read-only dispatch.** `nohell/SKILL.md` is 16 KB and
`HELL-CATALOG.md` behind it is 187 KB. A task that only greps, counts or reports pays that and
returns nothing for it. Attach when the dispatch writes code, changes a stored procedure, or
reviews someone else's work; skip it when the dispatch only reads.

`ponytail` composes with it: `ponytail` asks whether the code needs to exist, `nohell` asks whether
it will become hell if it does. A dispatch that builds something new can name both — in that
order, which is the order `nohell`'s own ladder already assumes it ran in.

## 5. Watch it without burning tokens

Dispatch, then arm one Monitor. It polls, prints only on a state change, and each event is a line:

**Verified working 2026-08-08** — this exact extraction was run against a live task and printed
`running 0 0` before being armed. Do not adapt it without re-running the inner `curl` once first.

```bash
D=/c/Users/Wassaphas/Desktop/cluade-orchestra
ID=<task id from the 201>
TOK=$(grep '^ORCHESTRATOR_TOKEN=' "$D/.env" | head -1 | cut -d= -f2- | tr -d '"'"'"' ')
prev=""
for i in $(seq 1 120); do
  cur=$(curl -s -m 10 -H "Authorization: Bearer $TOK" http://127.0.0.1:4123/tasks \
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
Monitor per task. The constraint is not the accounts, it is git: **six sessions on one worktree
share one staging area**, and two have already buried each other's work here. Give each its own.

⚠ **THERE ARE ALREADY FOURTEEN WORKTREES. Run `git -C C:/mm-t worktree list` BEFORE creating one.**
Measured 2026-08-12: `mm-a b c e f g h i j q r s v` plus `mm-t` (main) and the Desktop folder. This
section used to name three, and a session that believes it will helpfully add `mm-d` next to
thirteen it never looked at. Most sit idle at a stale commit on a branch named after whatever the
run was (`work/tax1`, `work/fifo1`, `work/zpark`) — **the branch name tells you nothing about what
is in it**, so check the tree, not the name.

Before reusing one, check BOTH, per worktree:

```bash
git -C C:/mm-<x> status --porcelain | wc -l          # uncommitted work — someone's, maybe live
git -C C:/mm-<x> rev-list --count main..HEAD         # commits not on main — unmerged work
```

`0` and `0` means free: `git -C C:/mm-<x> reset --hard main` and dispatch into it. **Anything else,
leave it alone and pick another.** On 2026-08-12 `mm-f` held 13 uncommitted files — a regenerated
SDK from a run the orchestrator's own crash had orphaned. Resetting it would have destroyed work
nobody had a record of, and the branch name (`work/x5`) said nothing about that.

⚠ **Never `git worktree remove --force` to clean up.** It follows the `node_modules` junction and
deletes the SHARED one in `C:/mm-t`, breaking every live worktree at once. Delete the junction
first. Reuse beats removal — that is why there are fourteen.

⚠ **The letter `t` is reserved.** `C:/mm-t` is main. Never name a worktree anything that a careless
glob or a tab-complete can turn into it.

Set up and verified 2026-08-09, `C:/mm-a`, `C:/mm-b`, `C:/mm-c`:

```bash
git -C C:/mm-t worktree add C:/mm-a -b work/a main
cp C:/mm-t/server/.env C:/mm-a/server/.env
```

```powershell
cmd /c mklink /J "C:\mm-a\server\node_modules" "C:\mm-t\server\node_modules"
cmd /c mklink /J "C:\mm-a\client\node_modules" "C:\mm-t\client\node_modules"
```

A junction rather than `npm ci` — the install is minutes per worktree and the trees are identical.
Confirm with `cd C:/mm-a/server && npx tsc --noEmit` before dispatching into it. Separate worktrees
have separate index files, which is what makes concurrent commits safe.

Tell each task which worktree is its own **and** that the others exist, or one will helpfully
"fix" a file another is mid-way through. Nothing enforces this but the prompt.

## 6-quater. ⚠ FAN-OUT HAS A DISK COST, AND IT IS THE ONE NOBODY BUDGETS FOR

**Check free space before dispatching, and again after.** `node_modules` is junctioned and costs
nothing per worktree — but **`.jest-cache` is not**, and every worktree that runs the gate builds
its own. Measured 2026-08-12 after twelve agents: `mm-t` 360 MB, and 13 more worktrees holding
44–144 MB each. Clearing the thirteen returned **1.21 GB**.

⚠ On that day the drive reached **4.8 MB free of 477 GB**, and the failure did not look like a full
disk. It looked like flaky tests:

```
ENOSPC: no space left on device, write        ← the only honest line, buried in one assertion
```

Everything else read as a timing problem — `backup-verify.test.ts` failing 2-of-13 then 4-of-13
then passing, taking 23–48 s instead of 7 s, `deploy-swap.test.ts` joining it, the
layering-ratchet ownership tests failing four assertions and passing on an immediate rerun. Two
sessions' worth of "the suite flakes under parallel load" was partly this. **A full disk presents
as nondeterminism, not as an error**, because only the tests that happen to write a temp file fail,
and which ones those are changes run to run.

So when several suites start flaking at once and the common factor looks like "load":

```bash
df -h /c | tail -1                                   # or Get-PSDrive C
for w in t a b c e f g h i j q r s v; do du -sm "/c/mm-$w/server/.jest-cache" 2>/dev/null; done
```

Jest caches are regenerable by definition — deleting them is always safe and always reversible.
Delete the worktrees' first and keep `mm-t`'s, since that is the one that runs the gate most.

⚠ Do NOT reach for `git worktree remove --force` to reclaim space. It follows the `node_modules`
junction and deletes the SHARED tree in `C:/mm-t`, breaking every live worktree at once.

## 6-ter. ⚠ NEVER PUT `git stash` IN A DISPATCHED PROMPT

**The stash stack is ONE per repository, not one per worktree.** It lives in `refs/stash` in the
common git dir, so every worktree pushes onto and pops off the *same* stack. Separate index files
do not save you here — that is the property that makes concurrent *commits* safe, and it does not
extend to the stash.

So `git stash push` … `git stash pop` — the obvious way to prove a test fails against the old code
— is a **cross-agent data race**. Nine agents doing it at once are popping each other's work off a
shared stack, LIFO, with no error and no warning.

⚠ This was discovered on 2026-08-12 the way these things usually are: `git stash list` in `mm-t`
showed `WIP on work/u5` and `WIP on work/zpark` — two *other* worktrees' agents, plainly visible
and poppable. A push/pop pair in `mm-t` happened to return the right stash only because nothing
interleaved in that second. And one agent had already **finished and left its stash behind**, so its
work was sitting orphaned on the shared stack.

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

**Dispatch more tasks than there are pools.** Six accounts, but the queue is real: excess tasks sit
`queued` and start as slots free. Nine concurrent were dispatched on 2026-08-12 across six pools and
the queue handled it without a 409. The number to size against is **free worktrees**, because two
tasks in one worktree share a staging area and will bury each other.

Skip `team1` (`claude.tsv`) when the user is actively typing to this session — it is usually their
own account and you are competing with them for it.

**Owner-relaxable, and it was relaxed on 2026-08-12** ("ยิงอัดได้เลยนะเกินกฎไม่เป็นไร ขยายกฎด้วย").
Volume is the owner's call because it is their budget. What is NOT relaxable, whatever the
instruction, and must stay in every dispatched prompt:

- no production database writes — port 3307 is an SSH tunnel to the live shop
- no raising a ratchet ceiling, no adding to an allowlist, no skipping a test, no `any`, no
  disabling a guard
- no push, no deploy, no dev server
- `git commit --only <paths>` — never `add -A`, never bare commit, never `reset --hard`

Those are safety, not convention. "Exceed the rules" means fan out wider, not ship unverified.

⚠ **Check the tunnel before writing the prompts, not after.** `(</dev/tcp/127.0.0.1/3307)` — if it
is CLOSED, every task must be scoped CODE-ONLY, and each prompt must say so and carry the figures
you already measured. Nine prompts that assume a live database against a closed tunnel is nine
wasted dispatches, and each one will spend its first minutes discovering the same thing.

**Put the refutation clause in every prompt.** "If the brief's premise turns out to be wrong, say
so with evidence and stop — a refuted premise is a good outcome." On 2026-08-12 that clause paid
three times: one agent proved a purchase-order VAT diagnosis wrong and found the real mechanism
(every PO carries a header rate of 7 because the create schema has no such field), which the
briefing session had got backwards.

## 6a. The bug that killed every long dispatch — fixed 2026-08-09, read this before "fixing" it again

Three sessions diagnosed this wrongly (twice as "trust", once as "a timeout") before anyone
looked at the database. **Diagnose from `permissions` and `task_events`, never from the error
string.** The string said `nothing on the stream`, which reads as a hang and was not one.

Two constants in that repo, each defensible alone, contradicted each other:

| file | value | what it did |
|---|---|---|
| `src/gate/permissions.ts` | **570s** | wait for a human to answer a permission card |
| `src/runner/input-limits.ts` | **300s + 30s** | shut a quiet pipe, then kill the child |

A run blocked on a card is quiet by definition, so the runner always shot first and the gate's
decision landed on a corpse. Five runs died at ~6 minutes, every one with its last stream event a
`tool_use` and a `permissions` row reading `status='denied', reason='no response'`, every one
retried 3-4× into the same wall. One was mid-refactor on `C:/mm-t` and left a half-applied edit
that broke that repo's typecheck for a day.

Fixed in `ac111d9`: decision 45s, silence budget 15 min, and an unanswered card now **allows**
(`ORCHESTRATOR_GATE_UNATTENDED=deny` restores blocking; `unattended` on `createGate` overrides per
gate). The row is still written with a reason naming the policy — `--ungated` would record nothing.

To read the evidence yourself, and note the Node version or `better-sqlite3` will refuse:

```bash
export PATH="/c/Users/Wassaphas/nodejs/node-v26.7.0-win-x64:$PATH"
cd /c/Users/Wassaphas/Desktop/cluade-orchestra
# tables: tasks, task_events, permissions, deploy_runs, pool_settings, thread_meta
```

⚠ **`cost_usd: null` does not mean nothing happened.** Cost is recorded from the final `result`
event, so a killed run reads null however much work it did. Task `394a7fc8` wrote nine files and
still shows null. Judge progress from `task_events`, not cost.

⚠ **A healthy running task has `duration_ms` and `cost_usd` null too**, so a watcher that prints
only on change will look identical whether the run is working or wedged. Check `task_events`
recency for a heartbeat.

## 6b. Two ways this was broken on 2026-08-08, by me

⚠ **NEVER dispatch a task that edits the orchestrator's own running code.** The control plane was
started with `npm run serve`, then given a task to fix its classifier, retry policy and preflight.
The server exited part-way through and every running task was orphaned — the queue lost their
results, though the files they had already written survived in the target repo.

The task that succeeded that day (`dc69f8b3`, $0.73, 112s) touched only `.env` and markdown. The
one that killed the server touched `src/`. That is the whole difference.

If the orchestrator must fix itself: drain the queue first, stop the server, dispatch from a second
copy, or run the target under `serve:watch` on a different port. Never edit `src/` of a live server
that is executing your own task.

⚠ **A watcher must distinguish "no answer" from "the answer is: finished".** Mine reported
`all terminal` when the server had simply gone down: curl returned nothing, the JSON parse threw,
the status string came back empty, the grep for `running` found nothing, and empty was read as
done. An earlier version watched the wrong endpoint entirely and would have stayed silent forever.

So every watch loop needs three outcomes, not two — **progressing**, **finished**, and
**cannot tell**. Make unreachable loud:

```bash
raw=$(curl -s -m 10 -H "Authorization: Bearer $TOK" http://127.0.0.1:4123/tasks) || raw=""
[ -z "$raw" ] && { echo "control plane unreachable — NOT a finished task"; exit 1; }
```

Silence and success must never render identically. That mistake was made three separate times in
one day, in three different scripts, each written by someone who had just documented the same trap
somewhere else.

## 7. When it misbehaves

That repository is the user's own and they are happy for it to be fixed rather than worked around.
It is TypeScript with vitest, `pnpm test` and `pnpm typecheck` at its root. Read `HANDOFF.md`
first — it carries the current state and the ordering hazards. Two known-wrong things are recorded
in §1 and §3 above; fix them at the source rather than teaching every caller to work around them.
