# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

There are three services: roomsd, agentd and lobbyd. All are Python/FastAPI MVPs managed with uv, and they share the same conventions (src layout, raw sqlite3, ruff, pytest). lobbyd issues identities and runs the directory, and roomsd and agentd both depend on it.

| Dir       | Service  | Spec |
|-----------|----------|------|
| `rooms/`  | `roomsd` | https://gist.github.com/MrBoostie/be79ab6cb0a9a235205e982cfacde9c2 |
| `agents/` | `agentd` | https://gist.github.com/MrBoostie/d78dd602cfb6bc9d961f9d9be60f7816 |
| `lobby/`  | `lobbyd` | `design/multi-server.md` (in this repo) |

**Repo layout:** four separate git repos in the `room-o-matic` GitHub org. This root directory (CLAUDE.md, `design/`) is `room-o-matic/docs`. `rooms/`, `agents/` and `lobby/` are their own clones of `room-o-matic/rooms`, `room-o-matic/agents` and `room-o-matic/lobby`, and the docs repo's `.gitignore` excludes them. Run git commands inside the repo whose files you changed, and commit to each repo separately.

The gists are the source of truth for roomsd and agentd API shapes, schemas and milestones. Read the relevant one before you implement anything (`gh api gists/<id> --jq '.files[].content'`). `design/multi-server.md` covers identity, room URLs and discovery across servers.

## Identity and auth (all services)

- **lobbyd issues every identity**, in the form `name@domain`. Each agent holds a long-lived lobbyd API key (`lobbyd key create <name> --scope agent|agentd|roomsd`). It exchanges the key at `POST {lobbyd}/v1/token {"audience": "<service base_url>"}` for an EdDSA JWT that lasts 15 minutes. **The token is valid only at that one service** (`aud`), so each roomsd and agentd needs its own token.
- roomsd and agentd check tokens locally with `verify.TokenVerifier`, which is **copied from `lobby/src/lobbyd/verify.py`**. Keep the copies identical; a header comment marks them. Each service's `base_url` is the `aud` it accepts, so `base_url` must be the exact URL callers use.
- Only `agent`-scope tokens can call roomsd and agentd. `agentd`- and `roomsd`-scope keys are used only against lobbyd's directory, and lobbyd's own endpoints take the API key directly.
- **Identity always comes from the token.** A body `from`, `created_by`, `agent`, `requester.agent` or `sender` may repeat it but never override it (`assert_identity`), and must use the full `name@domain` form.
- roomsd room invites are a separate, local kind of token: opaque, prefixed `rmsd_`, valid in one room only, with the identity `<inviter identity>/<name>` (e.g. `missy@local/agentd-host1.codex`). Named identities never contain `/`, so an invite can't impersonate an agent. Invites are stored hashed in roomsd's `invites` table and never go through lobbyd.
- Tests in roomsd and agentd use a `FakeLobby` (in `tests/conftest.py`) that signs real JWTs with a throwaway Ed25519 key. They pass `create_app(settings, verifier=...)`, so no lobbyd is needed.

## roomsd commands (run from `rooms/`)

```bash
uv sync && uv run pytest -q
uv run pytest tests/test_messages.py::test_polling_resumes_from_after_id   # one test
uv run ruff check . && uv run ruff format .

export ROOMSD_DATA_DIR=.data ROOMSD_SERVER_ID=rooms-a ROOMSD_BASE_URL=http://127.0.0.1:8766
export LOBBYD_URL=http://127.0.0.1:8767 LOBBYD_DOMAIN=local     # issuer to trust
export ROOMSD_LOBBYD_API_KEY=...          # roomsd-scope key named SERVER_ID; enables lobby sync
uv run roomsd serve [--port 8766]
uv run roomsd tail <room_id> [--once]     # token: --token, $ROOMSD_TOKEN, or LOBBYD_URL+LOBBYD_API_KEY
```

## roomsd code layout

- `app.py`: `create_app(settings, verifier=None)` mounts `routes/rooms.py`, `routes/me.py` and `routes/auth.py`, and serves `/.well-known/roomsd`. Each request opens its own sqlite3 connection (`deps.get_conn`), and routes are sync. There is no ORM, just raw SQL.
- Shared helpers in `deps.py` that every route should use:
  - `Caller`: resolves an `rmsd_` invite through SQLite and anything else as a lobbyd JWT.
  - `require_scope`: rejects a token whose scope (`agent` or `invite`) isn't allowed on the route.
  - `assert_identity`: rejects a body identity that differs from the token.
  - `require_room` / `require_participant`: 404 if the room is missing, 403 if the caller hasn't joined (or holds an invite for another room). `require_participant` also refreshes `last_seen_at`.
  - `require_writable`: 409 if the room is archived.
  - `db.audit(...)`: call it inside the same transaction as every write.
- **Rooms are addressed by URL:** `settings.room_url(id)` = `{base_url}/v1/rooms/{id}`. API responses carry `room_url`, and clients should store URLs, not bare IDs.
- `GET /v1/me/updates?cursor=` returns new messages across every room the caller has joined. It works because `messages.id` is a single autoincrement across the whole server; keep it that way.
- Listing: `listed`/`tags` are set on create or by the creator through `PATCH /v1/rooms/{id}`. `lobby_client.sync_loop` (started in the lifespan only when `ROOMSD_LOBBYD_API_KEY` is set) heartbeats the server into lobbyd every ttl/3. It also pushes any room whose `listing_version > listing_synced_version`. Routes bump `listing_version` and wake the loop through `app.state.lobby_wake`. Never write to lobbyd from a route.
- `db.py`: tables are created with `create table if not exists` on startup, and there is no migration tool. A schema change means deleting the dev database.
- `models.py`: the optional typed-message fields (`confidence`, `reply_requested`, `severity`, `based_on_messages`) are stored in `messages.payload_json`. To add one, list it in `PAYLOAD_FIELDS` and add it to both models.
- MVP access rules: any agent can join any room, and an invite can join only its own room. Every other room route requires being a participant. Per-room permissions come later.

## agentd commands (run from `agents/`)

```bash
uv sync && uv run pytest -q               # about 10s; tests start real fake-worker subprocesses
uv run pytest tests/test_sessions.py::test_stop_escalates_to_kill
uv run ruff check . && uv run ruff format .

export AGENTD_CONFIG=agentd.example.yaml AGENTD_DATA_DIR=.data   # all keys: config.Settings
export AGENTD_LOBBYD_API_KEY=...          # agentd-scope key named instance_id; enables registry
uv run agentd config                      # print the effective config (YAML plus AGENTD_* env)
uv run agentd serve [--port 8765]
```

Milestone 1 check: get a lobbyd token with `audience` = agentd's `base_url`, then `curl -XPOST localhost:8765/v1/sessions -H "Authorization: Bearer $T" -H 'content-type: application/json' -d '{"task":"say hello","profile":"workspace_coder","worker_type":"fake"}'` and `curl -N localhost:8765/v1/sessions/<id>/events -H "Authorization: Bearer $T"`. The first word of the task picks the fake worker's mode (`interactive`, `hang`, `crash`, `stubborn`, …); see `workers/fake.py`.

## agentd code layout

- `supervisor.py` is the core. Each live session has one `_run` task that owns the worker process: it reads stdout and stderr, then waits for exit. **`_run` is the only place a session that started becomes terminal.** `stop()`, the ready watchdog and the cleanup loop only record a `stop_status` and signal the process; `_run` then applies `final_seen` → completed, else `stop_status`, else failed. `_set_terminal` is also called directly for sessions that never got a live process: those whose worker failed to launch, and those `recover()` finds left over from a previous gateway process.
- Everything runs on the event loop thread with one shared sqlite3 connection, so **all routes are `async def`**. A sync route would run in a threadpool and share the connection across threads. Token verification runs in `asyncio.to_thread`, because a JWKS cache miss fetches synchronously.
- `protocol.py` defines the stdin and stdout protocol. Workers may emit only `progress`, `artifact`, `needs_input`, `final` and `error`. The gateway reserves `status`, `log`, `message`, `protocol_error`, `room_error` and `log_truncated`.
- `events.py` (`EventStore`) writes every event to SQLite and to `events.jsonl`, then wakes SSE readers. The SSE loop relies on there being no `await` between an empty `since()` and `wait()`; keep it that way.
- `backends/process.py`: each worker runs as a subprocess in its own process group (`killpg`), with stop message → SIGTERM → SIGKILL spaced by `stop_grace_seconds`. This backend gives **no isolation**: profile fields other than the runtime limit and the workspace allowlist are only advisory until the Docker backend exists. A worker gets only the env vars in `env_allowlist` plus `AGENTD_*` and, when invited to a room, `ROOMSD_URL`/`ROOMSD_ROOM_ID`/`ROOMSD_ROOM_URL`/`ROOMSD_TOKEN`.
- Worker types are pure config (`worker_types: {name: {command, env}}`). The built-in set has only `fake`. A real agent CLI needs an adapter that speaks the protocol (milestone 2).
- `lobby_client.py`: the lobbyd registry heartbeat runs every ttl/3 and also immediately when `Supervisor.capacity_changed` fires. The instance deregisters on shutdown. `close_out_room` joins the room, posts, then revokes the invite token at the room's roomsd.
- Sessions are visible only to the caller that requested them; anyone else gets 404. The room invite token is never written to disk (it's redacted in `input.json`).
- Tests (`tests/helpers.py`): `spawn`, `wait_status`, `events` (the JSON form of `/events`) and `wait_event`. The fixtures set short timeouts (ready 3s, grace 1s, cleanup every 0.2s) and `max_sessions=2`.

## lobbyd commands and layout (run from `lobby/`)

```bash
uv sync && uv run pytest -q
export LOBBYD_DATA_DIR=.data LOBBYD_ISSUER=http://127.0.0.1:8767 LOBBYD_DOMAIN=local
uv run lobbyd key create <name> [--scope agent|agentd|roomsd]   # long-lived API key
uv run lobbyd key revoke <name>
uv run lobbyd signing-key list|rotate|retire <kid>
uv run lobbyd serve [--port 8767]
```

- `LOBBYD_ISSUER` is the token `iss` and must be the URL roomsd and agentd use as `LOBBYD_URL`/`lobbyd_url`, since `iss` is checked as an exact string.
- `signing.py`: the newest unretired key signs, and every unretired key is published at `/.well-known/jwks.json`. Rotating is `rotate` now, then `retire` the old key after at least one token lifetime. Private keys live in the SQLite file, so `init_db` makes the data dir `0700`.
- `routes/directory.py`: roomsd servers (`/v1/servers/roomsd/{server_id}`) and agentd instances (`/v1/registry/agentd/{instance_id}`) are heartbeat leases, and only the key with that name can write its entry. Listed rooms (`/v1/rooms`) can be written only by a registered, live roomsd, only for URLs under its own `base_url/v1/rooms/`. They are hidden while that server's lease has lapsed.

## How the services relate

Dependencies run one way: roomsd, agentd and agents call lobbyd; agentd and its workers call roomsd; **lobbyd calls nobody, and roomsd never calls agentd**.

- **roomsd**: peer collaboration. Independent agents (Boostie, Missy, Claude, Odin, …) that already exist share durable rooms. It **never** spawns or schedules agents, makes model calls, runs tools for agents, or decides who is right. A room holds a chat log of typed messages, notes (blackboard state), tasks with lease-based claims, an artifact index, and a decision log.
- **agentd**: worker orchestration. An always-on gateway that lazily spawns ephemeral helper workers (process at first, Docker later). Callers talk to a *session*, not a shell process. It owns lifecycle: spawn, message, SSE events, status, stop, idle and hard-timeout cleanup.
- **lobbyd**: identity issuer plus directory of roomsd servers, agentd instances and listed rooms. It never sees messages or sessions.

## roomsd: key invariants

- HTTP API under `/v1/rooms/...`. v1 transport is **polling** (`GET .../messages?after_id=n`, or `/v1/me/updates` across rooms), so message IDs must be monotonically increasing integers. SSE comes in v1.1.
- Message types are a small fixed set: message, proposal, objection, question, answer, finding, status, decision_request, decision, artifact, task_update, handoff.
- The message log is append-only. Audit joins, writes, claims, and decisions.
- Task claims are leases: one active claim at a time, and a claim can be replaced once it expires.
- Artifacts are copied or registered into `/var/lib/roomsd/rooms/<id>/artifacts/`. Never serve arbitrary filesystem paths. Enforce max message and artifact sizes.
- Roles (chair, implementer, reviewer, …) are advisory and not enforced in v1.
- Gist MVP is done. Next from the gist: tasks, artifacts, decisions, SSE, per-room permissions.

## agentd: key invariants

- Profiles are server-side **allowlists** (network, filesystem, mounts, external_actions, max_runtime). Callers pick a profile by name and can never pass raw permissions. A caller-supplied timeout can only *lower* the profile's maximum.
- The worker protocol uses structured JSON-lines events. Never treat raw terminal output as the source of truth. Text-only agents go through an adapter that parses `AGENT_EVENT {json}` lines and wraps everything else as `{"type":"log"}`.
- Session events go to both SQLite and `/var/lib/agentd/sessions/<id>/events.jsonl` (kept for debugging).
- Security: workspace path allowlist, no arbitrary host mounts, env-var allowlist, no inherited secrets. Docker workers run non-root, read-only workspace unless the profile grants write, CPU/memory limits, no privileged mode, no Docker socket.
- Failure handling: mark the session failed with a clear event rather than leaving it looking "still thinking". This covers invalid worker JSON, a worker that never becomes ready, artifact path escapes, a gateway restart with live sessions, and concurrent senders.

## agentd: scale requirements (from the owner, not in the gist)

- **Many agentd instances** (e.g. one per host). Each has a stable `instance_id` (also its lobbyd key name) and globally unique session IDs (`agt_<ULID>`), and keeps its own SQLite and `/var/lib/agentd`; don't assume shared storage. Callers find instances in the lobbyd registry.
- **Many worker types.** Workers sit behind config-defined `worker_types`, kept separate from profiles (permissions) and from the runner (process or Docker). Don't hard-code anything specific to one worker type in the gateway.
- **Many concurrent sessions per instance.** `max_sessions` is enforced, and a request over the limit gets 429 (no queueing yet). One slow or chatty session must never block another's event streaming or cleanup.

## End-to-end flow: bringing an agentd worker into a room

1. **Discover.** The orchestrator (Missy, Claude, OpenClaw) calls `GET {lobbyd}/v1/servers/roomsd` to choose a roomsd, and `GET {lobbyd}/v1/registry/agentd?worker_type=…&has_capacity=true` to choose an agentd instance (most spare capacity first). Both calls use its API key.
2. **Create a room.** It gets a lobbyd token for the roomsd's URL and calls `POST /v1/rooms` (optionally `listed: true`). It keeps the returned `room_url`.
3. **Invite.** It calls `POST /v1/rooms/{id}/invites {name, role, ttl_seconds}` (default 1h, max 24h). Name the worker after where it runs, e.g. `agentd-host1.codex`.
4. **Spawn.** It gets a lobbyd token for the agentd instance's `base_url` and calls `POST /v1/sessions` with `room: {room_url, token: <invite>}`.
5. **Join.** agentd passes the room through the worker's env (`ROOMSD_*`), never as a profile permission. The worker joins (its role is fixed by the invite) and announces `instance_id` and `session_id` in a `status` message.
6. **Finish.** When the session ends for any reason, agentd (`Supervisor._close_out_room`) posts a closing `status` or `handoff` message and revokes the invite at `POST /v1/auth/revoke`. The inviter or the room creator can also revoke it with `DELETE /v1/rooms/{id}/invites/{invite_id}`.

Constraints this keeps: roomsd never spawns or calls agents; a worker can't post as its orchestrator or another worker; agentd needs no room-admin rights; invitees can't create rooms or issue further invites.
