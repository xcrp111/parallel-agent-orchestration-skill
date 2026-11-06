# Dispatch adapters

The core skill asks for a **handle**, a **status check**, and a **result**. This file maps those to concrete hosts. Probe before using; never invent a command.

## Tiers

| Tier | Contract | Survives this turn? | You must manage |
|---|---|---|---|
| A. Detached session | returns an id | yes | collect, stop, clean up |
| B. In-turn subagent | returns a result now | no | nothing (it blocks) |
| C. Detached process | returns a pid + log | yes | pid, timeout, log, cleanup |
| D. Sequential | none | — | say so |

Use A when available, B only for short work, C when the host has no native detach, D otherwise.

## Claude Code (verified against 2.1.284)

The stock CLI has a native background session. This is the portable path for a local Claude Code install.

```bash
# dispatch: the prompt is positional; --bg conflicts with --print
claude --bg "Read /abs/path/to/brief.md and execute it. Write the shard to /abs/path/to/shard.md before finishing."

# status: JSON array, scriptable
claude agents --json

# inspect / manage
claude logs <id>
claude attach <id>
claude stop <id>
claude rm <id>
```

`claude agents --json` fields observed on 2.1.284: `id`, `cwd`, `kind`, `startedAt`, `sessionId`, `name`, `state` (`blocked` is possible). Treat the schema as version-dependent.

**Operational notes:**

- The workspace must be trusted once: run `claude` interactively in that directory and accept the prompt, then retry `--bg`.
- A background session can stop at an approval prompt; `claude agents --json` then shows `state: blocked`. Pre-approve the tools it needs with the permission mode your policy allows, or `claude attach <id>` and answer. It will not silently proceed.
- `--bg` conflicts with `--print`; the CLI explains this and tells you to pass the prompt positionally.
- Background sessions can run in their own git worktree. Their edits may not be in your working tree; collect the result or merge the worktree before assuming the files changed. Verify this per version.
- `/subtask` is **not** part of stock 2.1.284. If a Cindy/plugin install adds it, treat it as Tier B, or Tier A if it accepts a background flag.
- The interactive Task/Agent subagent tool is Tier B: useful for a quick independent check, but it blocks the turn. Do not describe it as background work.

**Why a brief file:** passing a path avoids shell-quoting a long prompt and works identically from PowerShell, bash, and the agent's own shell tool. The worker reads the file; the shard path is absolute.

## DSH

- `subagent` starts a child, returns an id immediately, and notifies on completion. Continue working; collect at a milestone. `send_message` steers a running child.
- `subagent_fork` seeds the child with the current conversation; use only when that context is truly needed.
- More than four streams, or a per-item pipeline: use `workflow`.
- Workers inherit the working directory; pass absolute paths for briefs and shards.
- If the child's sandbox is read-only, it returns the shard body; the Lead writes it with `written_by: lead`.
- Skills for DSH live in `~/.agents/skills/`.

## Cindy

- Prefer its Lead/Worker sessions or a dragged-out tab if that gives you a handle. The exact dispatch API is **unverified**; do not assume one.
- If Cindy exposes `/subtask` in its Claude Code integration, Tier B (or A if background is supported); otherwise fall back to `claude --bg`.
- If Cindy runs DSH underneath, use `subagent`.
- Do not claim you opened a window. Cindy's window is a view; your job is to return the session.

## Any CLI (Tier C)

Works with `claude -p`, `codex exec`, `dsh`, or any agent that accepts a prompt and exits.

1. Write the brief to `<run>/briefs/<worker-id>.md`.
2. Launch detached, redirect output to `<run>/logs/<worker-id>.log`, record the pid.
3. At milestones, check the pid and tail the log.
4. On completion, read the shard; on failure, keep the log and mark the shard `failed`.
5. Stop or kill abandoned workers; remove temp worktrees.

POSIX:

```bash
nohup <agent> -p "$(cat brief.md)" > worker.log 2>&1 &
echo $! > worker.pid
```

PowerShell:

```powershell
$p = Start-Process -FilePath <agent> `
  -ArgumentList '-p', (Get-Content -Raw brief.md) `
  -RedirectStandardOutput worker.log -PassThru
$p.Id | Set-Content worker.pid
```

Exact flags differ per CLI. If the CLI has no non-interactive mode, it is not Tier C; use a session/tab or go sequential.

## Cross-host memory

- One canonical root: `PARALLEL_MEMORY_ROOT` if set, otherwise `<workspace>/.agents/memory/parallel`. Both Claude Code and DSH see the same absolute path.
- Skills: keep one copy in `~/.agents/skills/parallel-orchestration` and link `~/.claude/skills/parallel-orchestration` to it. On Windows this machine already uses directory junctions for exactly this pattern; on Unix use `ln -s`. One source of truth, every host.
- A shard written by Claude Code must be readable by DSH and vice versa: no host-specific fields, no absolute paths in the body, no secrets.
- If the two hosts run in different workspaces, point both `PARALLEL_MEMORY_ROOT` at one absolute directory.

## Capability matrix

| Host | Detach | Status | Steer | Cleanup |
|---|---|---|---|---|
| Claude Code 2.1.x | `claude --bg` | `claude agents --json` | `claude attach` | `claude stop` / `rm` |
| DSH | `subagent` | completion notice | `send_message` | cancel/interrupt |
| Cindy | Worker/tab or `/subtask` | host UI | host UI | host UI |
| Generic CLI | background process | pid + log | signal / stdin | kill |

## Failure modes

| Failure | Response |
|---|---|
| Claude Code: workspace not trusted | Run `claude` interactively once, accept trust, retry |
| Claude Code: `--bg` + `--print` conflict | Drop `--print`; pass the prompt positionally |
| Claude Code: session `blocked` | `claude attach <id>` or `claude logs <id>`; do not dispatch a duplicate |
| Claude Code: edits invisible in main tree | The session may use a worktree; merge or collect it explicitly |
| DSH: child read-only | Child returns shard body; Lead writes |
| Generic: no pid | Not detached; treat as Tier B/D |
| Any: no handle | Sequential; no fake parallelism |
