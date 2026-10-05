# dispatchd: scheduled and webhook-triggered rooms

Status: implemented (room-o-matic/dispatch). Running a room by hand means creating it, inviting agents and writing the goal. dispatchd does all of that on a schedule, or when another system calls a webhook.

## Placement and trust

dispatchd is a **separate service**, not part of lobbyd. lobbyd's rule is that it calls nobody: it issues identities and keeps the directory. A dispatcher has to *act*: create rooms, summon workers, send offers. That needs an acting identity, which must not sit next to lobbyd's signing keys.

dispatchd is an **ordinary agent identity** (e.g. `dispatch@domain`) and acts through the `roomomatic` client like any orchestrator. No service gives it special powers:

- **agentd** decides which profiles and worker types it may run (`callers` policy). A template can ask for less, never more.
- **roomsd** applies each room's admission and rights. dispatchd is the creator, and so the admin, of the rooms it opens, and nothing else.
- **lobbyd** delivers its offers to named peers. Offers grant nothing; dispatchd grants each peer rights in its own closed room first.

Dependencies still run one way: dispatchd calls lobbyd, roomsd and agentd, and nothing calls dispatchd except operators and webhook callers.

## Templates are the restriction

A template fixes:

- **The room:** admission (closed by default), listing (off by default), `max_hops`, message rate, and how long before the room is archived.
- **Workers:** agentd `worker_type` and `profile`, and optionally a `workspace` (a directory on the agentd host, e.g. a repo used read-only as a knowledge base).
- **Peers:** each one's rights and role.
- **The run:** `max_duration`, the overlap policy, and `order` (`parallel`, or `sequential` so each worker builds on the previous one's posts).

Schedules and webhooks name a template with `use:`.

| | enforced by | example |
|---|---|---|
| what a worker may do | agentd profile, and agentd's grant to dispatchd | `read_only_research`: no workspace, read-only |
| who may read or write the room | roomsd admission and rights | closed room; the peer gets `[read, write]` |
| how much the room may churn | roomsd loop guards | `max_hops: 3`, `message_rate_per_minute: 30` |
| how long and how much | agentd runtime and budgets; dispatchd `max_duration` | stop at 1 h |
| `rules` | **nobody**: guidance text in each task and in the `restrictions` note | "describe changes, don't make them" |

## A run

The steps are the same for every trigger (`schedule`, `webhook`, `manual`). Each is recorded as it completes, so a run interrupted by a restart resumes without duplicates. Worker summons use `operation_id = <run>.<worker>`, and peer offers use `offer_id = <run>.<agent>`.

1. Create the room and apply its guards.
2. Seed the `goal` and `restrictions` notes, and post the goal (topic `goal`). The post names the worker handles: each worker's name plus the run's short id, so overlapping runs never share a guest identity.
3. Grant each peer its rights, then send its offer.
4. Summon the workers: all at once, or one after another with `order: sequential`.
5. Wait for every worker session to end, or stop the rest at `max_duration`.
6. Call `finalize_session` for any room cleanup agentd handed back.
7. Mark the run `done` or `failed`. Archive the room after `archive_after`.

Oneshot worker types end a run when they finish; interactive ones keep the room open until `max_duration`.

## Schedules

- **Cron:** five fields (minute hour day month weekday), evaluated in the schedule's time zone. A built-in parser handles DST, and cron that never fires is rejected.
- **First sight and edits:** a schedule is scheduled when first seen, not fired, and an edit to its cron or time zone recomputes the next fire.
- **Missed fires:** a fire missed by more than `catch_up` (dispatchd was down) is recorded as a skipped run and not replayed.
- **Overlap:** `skip`, or `allow` up to `max_concurrent`.

## Webhooks

`POST /v1/hooks/<name>` must carry these headers:

- `X-Rom-Timestamp`: Unix seconds; refused if more than 5 minutes off.
- `X-Rom-Signature: sha256=<hex HMAC-SHA256(secret, "<timestamp>.<raw body>")>`
- `X-Rom-Delivery`: an id that deduplicates retries.

The body is `{"prompt", "goal"?}` only; unknown keys are refused, so a caller can't try to override the template. The prompt is untrusted and only appears inside the operator's `task_template`. The body size is capped before parsing, and each webhook has an hourly rate limit, though retries of a known delivery are still answered. Callers poll `GET /v1/hooks/<name>/runs/<id>`, signed the same way.

There are **no outbound callbacks** (an SSRF risk). Results live in the room, and callers poll the run status.

## Operating it

- **Definitions** live in an operator YAML file, hot-reloaded. An invalid edit keeps the last good definitions running and shows in `/readyz`.
- **Operator API:** lobbyd tokens from identities in `DISPATCHD_OPERATORS` (default deny) can list runs and schedules and trigger a schedule. lobbyd mints tokens only for approved endpoints, so dispatchd's URL is approved with an inert `service` key: `lobbyd key create dispatchd --scope service --endpoint <url>`.
- **Definitions from the API:** operators (and Claude Code through `rom mcp`) can also create templates, schedules and webhooks over the operator API. They are stored in dispatchd's database and validated **with the config file as one set**: an unknown template, a bad cron or deleting a template in use is a 422. The file stays authoritative: its definitions are read-only over the API (409), and a name in both is an error. A colliding file edit keeps the last good set and fails `/readyz`.
- **API webhook secrets** are generated by dispatchd (`whsec_…`) and returned only when the webhook is created or its secret rotated. They can't name a `secret_env`. A retired secret (after a rotation or a deletion) is journaled by fingerprint before the change commits, and a restore clears any stored secret the journal lists, so an old snapshot can't revive it.
- **Health:** `/readyz` checks the database, the config and the scheduler loop; `/metrics` reports runs by state, webhook volume and scheduler health.
- **Backups:** the shared `ops.py` versions the schema, backs up and restores. On restore, unfinished runs are marked failed rather than resumed (see `design/operations.md`).
