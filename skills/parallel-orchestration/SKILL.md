---
name: parallel-orchestration
description: Use when a task is long-running or contains two or more independent workstreams, or when the user asks to run work in parallel, keep working in another window or tab, dispatch background subagents, or merge parallel results into shared memory; also use before spawning multiple agents against one repository. Works in Claude Code, DeepSeek Harness, Cindy, and any other host that loads SKILL.md.
whenToUse: Use when the user says "并行", "同时做", "开另一个窗口", "后台跑", "split this up", "work on something else meanwhile", "dispatch subagents", or when a multi-module/multi-question task is expected to run longer than about ten minutes. Works with Claude Code (including stock 2.1.x via `claude --bg`), DSH, and Cindy. Not for single-file fixes, one blocking dependency chain, or work that needs continuous human judgment.
---

# Parallel Orchestration

**Core principle:** parallelism is not "more agents". It is partitioning work into independently verifiable streams with disjoint write scopes, then merging conclusions into a shared, harness-neutral memory contract.

A forked worker inherits context **at dispatch time**. It does not see later turns, and its context dies with its session. The Markdown shard is the only memory that survives across windows, hosts, and Harnesses.

**Portability contract:** the core loop never names a host command. It asks for a *handle*, a *status check*, and a *result*; the adapter supplies concrete commands. The same skill must run unchanged in stock Claude Code, DSH, Cindy, and a plain CLI agent.

## 0. Probe before planning (once per session)

Do not assume a command or path exists. Check, then record the answers in the run ledger.

| Question | How to check | If absent |
|---|---|---|
| Which host is running? | Claude Code, DSH, Cindy, another SKILL.md host | Use the generic adapter |
| Can it detach work from this turn? | Claude Code 2.1.x: `claude --bg "<brief>"`; DSH: `subagent`; host Worker | Use a detached CLI process (Tier C) or go sequential |
| Can it report status later? | Claude Code: `claude agents --json`; DSH: completion notification; process: pid + log | Poll the log at milestones only |
| Where is shared memory? | `PARALLEL_MEMORY_ROOT` → host's documented shared-memory directory → `<workspace>/.agents/memory/parallel` → ask | Ask; never guess silently |
| Can a worker write files? | Sandbox/permission policy | Worker returns the shard body; Lead writes it |

Never tell the user you opened an OS window. A new window or dragged-out tab is a host UI affordance; your job is to keep the session free by dispatching background work.

## 1. Gate: split, offload, or stay sequential?

**Split mode (2–4 workers)** — all four must hold:

1. The task will run longer than roughly ten minutes, or has at least three tool-heavy phases.
2. There are at least two workstreams with **separate outputs and separate acceptance checks**.
3. Write scopes are disjoint, or the outputs are read-only and merge trivially.
4. The Lead has useful non-overlapping work while workers run.

**Offload mode (1 worker)** — all three must hold:

1. The task is long but self-contained enough to hand off with a brief.
2. The user wants the session free (the "another window" case), or the Lead would otherwise idle.
3. The result can be merged by reading one shard; it does not need continuous back-and-forth.

Offload is not parallelism, but it satisfies the real request: the long task keeps running while the user does something else. It is also the correct fallback when a task cannot be split. It requires Tier A or C; an in-turn subagent (Tier B) cannot offload because it blocks.

**Stay sequential** if any blocker applies:

- one blocking dependency chain and no self-contained unit to hand off;
- two workers would edit the same file or shared mutable state;
- the work needs continuous human judgment or approval at every step;
- the task involves credentials, destructive commands, or irreversible operations without an explicit approval path;
- the only goal is "use more agents".

If the gate fails, say so in one line and continue sequentially. Do not dispatch a worker just to look busy.

**Special case — user asks for another window.** Prefer split mode if independent streams exist; otherwise offload the whole task as one worker. If it cannot be handed off, say so and continue. Never claim to have opened an OS window.

## 2. Plan before dispatching

Keep it to five minutes and write it down.

- **1–4 workers.** Split mode uses 2–4; more than four means coordination cost exceeds the gain — batch the work or use a workflow engine. Offload mode uses exactly one.
- For each worker define: `id`, `goal`, `inputs` (paths/URLs, not pasted content), `write_scope`, `done_criteria`, `timebox`, `report_format`.
- Draw the dependency DAG. Dispatch only roots; dependents wait for their inputs.
- Reserve all integration files (`index.md`, lockfiles, shared schemas, migration order) for the Lead.
- State the plan to the user in three bullets: who does what, what the Lead does, when to expect results. Mention the worker count and that it costs extra tokens.

Write `plan.md` under the run directory before the first dispatch:

```
<memory-root>/runs/<run-id>/plan.md
```

`run-id` = `YYYYMMDD-HHmm-<slug>` (local time), e.g. `20261105-2230-auth-refactor`.

## 3. Worker brief

Use [`references/worker-brief.md`](references/worker-brief.md) as the starting point and fill every field.

- Give **pointers**, not dumps: file paths, commands to run, the decision already made.
- State what the worker must **not** touch.
- Require an output contract: summary ≤ 10 lines, files changed, verification performed, open questions, shard path.
- Require the worker to write its own shard **before** it finishes, at the absolute path you give it.
- Never send "continue my work" without acceptance criteria.

## 4. Dispatch by capability, not by host name

Pick the highest tier the host supports, record the handle, then continue.

| Tier | What you get | Concrete examples | Trade-off |
|---|---|---|---|
| **A. Detached session** | an id now; survives this turn and the app | Claude Code `claude --bg "<brief>"`; DSH `subagent`; Cindy Worker | Preferred |
| **B. In-turn subagent** | a child result, returned in this turn | Claude Code Task/Agent tool; `/subtask` if a plugin installs it | Not background; short work only |
| **C. Detached OS process** | a pid and a log file | `claude -p`, `codex exec`, or `dsh` launched detached | You own status, timeout, cleanup |
| **D. Sequential** | no handle | — | Say so; never fake it |

**Claude Code (stock 2.1.x):**

- `claude --bg "<brief>"` returns an id. `--bg` conflicts with `--print`; pass the task positionally.
- Status: `claude agents --json`. Logs: `claude logs <id>`. Attach: `claude attach <id>`. Stop: `claude stop <id>`. Remove: `claude rm <id>`.
- The workspace must be trusted once by running `claude` interactively and accepting the prompt.
- `/subtask` is **not** in stock Claude Code 2.1.284. If your Cindy/plugin setup adds it, treat it as Tier B, or Tier A if it accepts a background flag.

**DSH:**

- `subagent` returns an id immediately; `send_message` steers a running child; completion is notified.
- `workflow` for more than four streams or per-item pipelines.
- Workers inherit the working directory; pass absolute paths.

**Cindy:**

- Prefer its Lead/Worker or tab-drag capability if it gives you a handle. Exact API is unverified; do not assume one.
- If it runs Claude Code underneath, Tier A via `claude --bg` is the fallback.

**Any CLI (Tier C):**

- Write the brief to `<run>/briefs/<worker-id>.md`.
- Launch detached with stdout/stderr redirected to `<run>/logs/<worker-id>.log`; record the pid.
- POSIX shape: `nohup <agent> -p "$(cat brief.md)" > worker.log 2>&1 & echo $!`
- Windows shape: `Start-Process -FilePath <agent> -ArgumentList ... -RedirectStandardOutput worker.log -PassThru`
- Check at milestones with the pid and the log tail; do not tight-loop.

Record for every dispatch:

| field | example |
|---|---|
| run_id | `20261105-2230-auth-refactor` |
| worker_id | `billing` |
| tier / host | `A` / `claude-code` |
| dispatch | `claude --bg "<brief>"` or `subagent` |
| handle | task id, session id, or pid |
| status_cmd | `claude agents --json` / notification / `ps -p <pid>` |
| write_scope | `src/billing/**` |
| status | `running` |
| started / timebox | ISO time / 25 min |

Append to `index.md` at dispatch, not after completion.

## 5. While workers run

- Do the Lead's own work first: integration, tests, the next dependency, the write-up. Do not redo a worker's scope.
- Check at natural milestones only — after finishing a unit of your own, or roughly every 10–15 minutes. No busy polling, no `sleep` loops. Use the host's status surface (`claude agents --json`, a DSH completion notice, or the pid/log for Tier C).
- Update the ledger whenever a state changes.
- If a worker asks a question, answer with pointers or a decision; do not take over its scope.
- If a worker exceeds its timebox, collect partial output and decide: extend once, narrow the scope, or cancel. Record the decision.

## 6. Collect and integrate

1. Verify against `done_criteria` yourself. "Done" from a worker is a claim, not evidence.
2. Review the diff for scope violations. Anything outside `write_scope` is a finding, not a bonus.
3. Resolve conflicts as the Lead. Never let two workers merge into the same file; if it already happened, stop, diff, and pick one owner.
4. Mark partial results `partial` and list the gaps. Do not silently fill them in and present a clean story.
5. Cancel or explicitly abandon workers that no longer matter; say which.

## 7. Write back to shared memory

This is the part that makes the next window useful.

Resolve the memory root once per session (see §0), then:

```
<root>/runs/<run-id>/
├─ index.md          # Lead owns; one row per worker + run status
├─ plan.md           # Lead owns; written before dispatch
├─ decisions.md      # Lead owns; choices that affect later work
└─ <worker-id>.md    # Worker owns; one shard per workstream
```

Use the schema in [`references/memory-shard.md`](references/memory-shard.md). Rules:

- The **worker writes its own shard** before reporting; the Lead only writes `index.md` and `decisions.md`.
- Append-only: never rewrite another worker's shard. Correct with a new entry plus a link.
- Self-contained: a reader in the other window or the other Harness must understand it without the chat log. This is what "shared history" means in practice — shared conclusions, not a shared transcript.
- ISO-8601 timestamps; repo-relative paths in `files`; link only artifacts inside the run directory; no secrets, tokens, or customer data.
- Keep each shard under one screen. Details go in the linked artifacts.
- If the worker cannot write, it returns the shard body and the Lead writes it under the worker's id, marked `written_by: lead`.

Update `index.md` when the run closes: final status, shard links, conflicts resolved, follow-ups.

## 8. Report to the user

One message, in the user's language:

```
Run <run-id>: 2 done / 1 running (~10 min left) / 0 blocked.
Done: billing API (shard), notifications queue (shard).
Lead: integration tests passing; auth module still running.
Decision needed: keep the old webhook retry policy? (decisions.md)
```

Then link the run directory. No raw logs, no play-by-play.

## 9. Failure handling

| Symptom | Action |
|---|---|
| No background handle | Do not fake it; use Tier C or run sequentially and say the host lacks detached dispatch |
| Claude Code: `Workspace not trusted` | Run `claude` once interactively in that workspace, accept the trust prompt, then retry `claude --bg` |
| Claude Code: `--bg and --print conflict` | Drop `--print`; pass the task positionally: `claude --bg "<brief>"` |
| Background session shows `blocked` | `claude attach <id>` to answer, or `claude logs <id>` to see the prompt; do not start a duplicate |
| Worker stalls past timebox | Collect partial, check shard, decide extend/narrow/cancel once |
| Two workers touched the same file | Stop integration; diff; assign one owner; record the conflict in `decisions.md` |
| Memory write fails | Lead writes the returned shard body; if still failing, put it in the workspace-local `.agents/memory/` and tell the user |
| Lead finishes first | Do not idle-poll; do the next dependency or prepare the integration; collect when notified |
| User interrupts | Cancel or park workers explicitly; write a `partial` shard so the next session can resume |
| Worker returns a novel plan instead of doing the task | Re-scope with a narrower brief; do not accept a plan as a deliverable |

## 10. Red flags

- "I'll open five agents to be safe." → More agents is not more throughput.
- "They can both edit this file; git will sort it out." → No. One writer per file.
- "I'll check status every 30 seconds." → Work instead; check at milestones.
- "The worker said it's done." → Verify the acceptance criteria.
- "I'll summarize in chat; memory can wait." → The chat dies with the session; write the shard.
- "I'll open a new window for you." → You cannot; dispatch background work instead.
- "The task is big, so it must be parallel." → Big and sequential is still sequential.

## 11. Checklist

Before dispatch:

- [ ] Gate passed (4/4), blockers absent.
- [ ] Mode chosen: split (2–4 workers) or offload (1 worker), with reasons.
- [ ] Every worker has goal, inputs, write scope, done criteria, timebox.
- [ ] DAG written; only roots dispatched.
- [ ] Lead has non-overlapping work.
- [ ] Memory root resolved; run directory created; `plan.md` written.
- [ ] User told the plan, worker count, and expected wait.

Before closing:

- [ ] Every worker has a shard (`done` / `partial` / `failed`).
- [ ] `index.md` and `decisions.md` updated.
- [ ] Conflicts resolved by the Lead; no shared-file writes.
- [ ] Verification run by the Lead, not just reported by a worker.
- [ ] Abandoned workers cancelled or explicitly parked.
- [ ] Final report links the run directory.

## References

- [`references/worker-brief.md`](references/worker-brief.md) — copy-paste brief with the output contract.
- [`references/memory-shard.md`](references/memory-shard.md) — `schema: 1`, index and decision templates.
- [`references/harness-adapters.md`](references/harness-adapters.md) — Claude Code / DSH specifics and known unknowns.
