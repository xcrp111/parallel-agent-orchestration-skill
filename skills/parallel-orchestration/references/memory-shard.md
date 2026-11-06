# Shared memory shards — schema 1

One run = one directory. One worker = one shard. The Lead owns the index and the decisions file.

This schema is the contract between the Skill and any future Plugin. Changing a field requires bumping `schema` and updating both sides.

## Layout

```
<memory-root>/runs/<run-id>/
├─ index.md          # Lead
├─ plan.md           # Lead, written before dispatch
├─ decisions.md      # Lead
├─ billing.md        # worker "billing"
└─ auth.md           # worker "auth"
```

`<memory-root>` is resolved per session: `PARALLEL_MEMORY_ROOT` → user-given path → host's documented shared-memory directory → `<workspace>/.agents/memory/parallel` → ask. Never write into the Skill directory.

Add `.agents/memory/` to the target project's `.gitignore` unless the team explicitly wants to commit shards.

## Worker shard

```markdown
---
schema: 1
run: 20261105-2230-auth-refactor
worker: billing
status: done            # done | partial | failed
harness: claude-code    # claude-code | dsh | other
started: 2026-11-05T22:31:00+08:00
finished: 2026-11-05T22:52:00+08:00
files:
  - src/billing/client.ts
  - tests/billing/charge.test.ts
tags: [billing, api-v2]
---

## Result

- Migrated all charge calls to v2; removed the v1 adapter.
- Billing suite: 42 passed, 0 failed.

## Changes

- `src/billing/client.ts` — v2 endpoint, idempotency key.
- `tests/billing/charge.test.ts` — updated fixtures.

## Verification

- `pnpm test billing` → 42 passed.
- `rg "chargeApiV1" src/billing` → no matches.

## Open questions

- Retry policy for 409 is unchanged; product may want backoff. Not blocking.

## Next for the Lead

- Regenerate the API docs from `client.ts`.
- Reuse the idempotency key helper in `notifications` if that worker needs it.
```

### Field rules

| Field | Required | Rule |
|---|---|---|
| `schema` | yes | integer; currently `1` |
| `run` | yes | same string as the directory |
| `worker` | yes | same string as the file name |
| `status` | yes | `done` / `partial` / `failed`; `partial` must list gaps |
| `harness` | yes | where it ran; needed to interpret tool behavior |
| `started` / `finished` | yes | ISO-8601 with offset; `finished` may be absent while running |
| `files` | yes | repo-relative; empty list allowed for read-only work |
| `tags` | no | short, lowercase, no spaces |
| `written_by` | no | `lead` when the worker could not write the file |

Body sections are required in this order. Keep the whole shard under one screen. Link to long artifacts instead of pasting them.

## Run index

`index.md` is the Ledger. Update it at dispatch and at close.

```markdown
# Run 20261105-2230-auth-refactor

- Goal: migrate auth, billing, notifications to the v2 charge API.
- Status: running
- Started: 2026-11-05T22:30:00+08:00
- Memory root: <root>

| worker | scope | status | started | shard |
|---|---|---|---|---|
| billing | `src/billing/**` | done | 22:31 | [billing.md](billing.md) |
| notifications | `src/notifications/**` | running | 22:33 | [notifications.md](notifications.md) |
| auth | `src/auth/**` | blocked | 22:35 | [auth.md](auth.md) |

## Integration

- [ ] billing merged
- [ ] notifications merged
- [ ] auth unblocked (waiting on schema decision)
- [x] full test suite run

## Follow-ups

- Regenerate API docs.
```

## Decisions file

Only decisions that affect later work. One line each, with the reason and the date.

```markdown
# Decisions — run 20261105-2230-auth-refactor

- 2026-11-05: keep the v1 webhook retry policy for now; v2 work is scoped to charges only. Reason: product has not signed off on backoff changes.
- 2026-11-05: `auth` stays single-threaded; it shares `src/schema.ts` with billing. Reason: avoid two writers.
```

## Hygiene

- Append-only. To correct a shard, add a dated entry at the bottom; do not rewrite history.
- No secrets, tokens, customer data, or raw logs.
- Write for a reader in another window, another Harness, or a week later.
- If a shard exceeds one screen, move detail into a linked file and keep the conclusion here.
