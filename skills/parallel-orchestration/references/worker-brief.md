# Worker brief template

Copy this into the dispatch call. Replace every `<...>`. Delete sections that truly do not apply; do not delete the output contract or the shard path.

---

## Role

You are the `<worker-id>` worker on run `<run-id>`. You own one workstream, not the whole task. The Lead handles integration and anything outside your write scope.

## Goal

`<one sentence describing the deliverable, not the activity>`

## Context you need

Read these first; do not ask for a summary of the repository.

- `<path/to/file>` — `<why>`
- `<path/to/test or doc>` — `<why>`
- Decision already made: `<decision + link>` (do not reopen it)

## Inputs

- `<absolute path / command / URL>`
- `<sample data or fixture>`

## Write scope

You may create or modify only:

- `<src/billing/**>`
- `<tests/billing/**>`

Do **not** touch:

- `<shared schema, lockfile, migration order, other workers' directories>`
- Anything outside this repository.

If you believe a change outside the scope is required, stop and report it instead of making it.

## Done criteria

- [ ] `<observable result 1>`
- [ ] `<test/command that must pass>`
- [ ] `<doc/API updated if applicable>`

## Timebox

`<25>` minutes. If you cannot finish, stop at a coherent point, mark the shard `partial`, and report what remains. Do not silently overrun.

## Output contract

Reply with exactly:

1. **Result** — ≤ 10 lines: what is now true that was not before.
2. **Files** — paths changed/created.
3. **Verification** — commands run and their outcome.
4. **Open questions / risks** — anything the Lead must decide (or `none`).
5. **Shard** — the absolute path you wrote (below).

## Shared memory shard

Before you finish, write this file:

`<memory-root>/runs/<run-id>/<worker-id>.md`

Use the schema from `references/memory-shard.md` (`schema: 1`). Include:

```yaml
---
schema: 1
run: <run-id>
worker: <worker-id>
status: done        # done | partial | failed
harness: <claude-code | dsh | other>
started: <ISO-8601>
finished: <ISO-8601>
files:
  - <relative path>
tags: [<short tags>]
---
```

Then the sections `## Result`, `## Changes`, `## Verification`, `## Open questions`, `## Next for the Lead`.

Write facts only. No transcript, no secrets, no "see chat".

If the filesystem is read-only for you, return the full shard body in your reply and say `shard_written: false`; the Lead will write it.

---

## Filled example (abridged)

```text
Role: billing worker on run 20261105-2230-auth-refactor.
Goal: migrate src/billing from the v1 charge API to v2 and keep the test suite green.
Context: src/billing/client.ts, docs/charge-api-v2.md, decision ADR-014 (v2 only).
Write scope: src/billing/**, tests/billing/**. Do not touch src/auth/** or pnpm-lock.yaml.
Done: all billing tests pass; no v1 imports remain; error mapping documented.
Timebox: 25 min.
Output: result / files / verification / open questions / shard path.
Shard: <root>/runs/20261105-2230-auth-refactor/billing.md
```
