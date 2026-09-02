# MACHINE.local.md — this machine's orchestra settings

Copy to `MACHINE.local.md` beside `SKILL.md` and fill in by **measuring**, not by copying another
machine's file. It is gitignored, so it never travels — that is the point. `SKILL.md` §0 reads it.

⚠ Never paste `ORCHESTRATOR_TOKEN` here. The skill reads it from `$ORCH/.env` at run time.

---

## Control plane

| field | value | how it was found |
|---|---|---|
| `$ORCH` | `<absolute path to the orchestrator checkout>` | dir whose `.env` has `ORCHESTRATOR_TOKEN` |
| `$PORT` | `4123` | default, unless `$ORCH/.env` overrides it |
| `$NODE` | `<dir containing the node binary>` | version whose ABI matches `better-sqlite3` |
| node version | `<vXX.Y.Z>` / ABI `<n>` | `node -p "process.versions.modules"` |
| has the §6a timeout fix? | yes / no / unchecked | read `src/gate/permissions.ts` + `src/runner/input-limits.ts` |

## Pools

From `GET /pools` on `<date>`. **Re-read rather than trusting this table** — it drifts.

| pool | account | notes |
|---|---|---|
| `<name>` | `<account>` | ⚠ the owner's own — skip it when they are typing to this session |
| `<name>` | `<account>` | |
| `<name>` | | scratch only, not a work pool |

Machine concurrency ceiling: **`<n>`** total agents (not per pool). Measured by: `<what broke>`.

## Target repos

| repo | main checkout | verify command | branch layout |
|---|---|---|---|
| `<name>` | `<path>` | `<exact command that must go green>` | `<main + worktree scheme>` |

Worktrees: `<naming scheme>`. Main is `<path>` — **reserved**, never let a glob reach it.
Shared `node_modules` junction lives in `<path>`; deleting a worktree with `--force` destroys it.

## Production hazards — name these in every dispatched prompt

- `<e.g. port NNNN is an SSH tunnel to a live database>` — check with `(</dev/tcp/127.0.0.1/NNNN)`
  before writing prompts; CLOSED means every task is scoped CODE-ONLY
- `<e.g. deploy target, prod connection string, anything a task must not touch>`

## Skills on this machine

| skill | absolute path to paste into prompts |
|---|---|
| `nohell` | `<path>/nohell/SKILL.md` |
| `ponytail` | `<path>/ponytail/SKILL.md` |
