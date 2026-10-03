# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

All three services (roomsd, agentd and lobbyd) are Python/FastAPI MVPs managed with uv, and they share the same conventions (src layout, raw sqlite3, ruff, pytest, the same key/token CLI). The two directories implement two separate design specs:

| Dir       | Service  | Spec |
|-----------|----------|------|
| `rooms/`  | `roomsd` | https://gist.github.com/MrBoostie/be79ab6cb0a9a235205e982cfacde9c2 |
| `agents/` | `agentd` | https://gist.github.com/MrBoostie/d78dd602cfb6bc9d961f9d9be60f7816 |
| `lobby/` | `lobbyd` | `design/multi-server.md` (in this repo) |

**Repo layout:** three separate git repos in the `room-o-matic` GitHub org. This root directory (CLAUDE.md and cross-service docs) is `room-o-matic/docs`. `rooms/`, `agents/` and `lobby/` are their own clones of `room-o-matic/rooms`, `room-o-matic/agents` and `room-o-matic/lobby`, and the docs repo's `.gitignore` excludes them. Run git commands inside the repo whose files you changed, and commit to each repo separately.

The gists are the source of truth for the API shapes, schemas, and milestones. Read the relevant one before you implement anything (`gh api gists/<id> --jq '.files[].content'`).

## roomsd commands (run from `rooms/`)

uv manages the project (Python ≥3.12, src layout in `src/roomsd/`).

```bash
uv sync                                   # install deps and dev tools
uv run pytest -q                          # all tests
uv run pytest tests/test_messages.py::test_polling_resumes_from_after_id   # one test
uv run ruff check . && uv run ruff format .

export ROOMSD_DATA_DIR=.data              # default is /var/lib/roomsd
uv run roomsd token create <agent>        # prints a bearer token (only its sha256 is stored)
uv run roomsd token create <instance_id> --scope agentd   # token for an agentd instance
uv run roomsd token revoke <agent>
uv run roomsd serve [--port 8766]         # agentd's spec uses 8765, so roomsd defaults to 8766
uv run roomsd tail <room_id> --token ... [--once]   # or set ROOMSD_TOKEN / ROOMSD_URL
```

## roomsd code layout

- `app.py`: `create_app(settings)` builds the FastAPI app and mounts the routers in `routes/` (`rooms.py`, `registry.py`, `auth.py`). Each request opens its own sqlite3 connection (`deps.get_conn`). There is no ORM, just raw SQL.
- Tokens have a **scope** (`auth.Principal.scope`), and every route must check it:
  - `agent`: a named agent (Boostie, Missy, …). Full access to rooms, subject to membership.
  - `agentd`: an agentd instance. It may only manage its own registry entry, where the token's agent name is its `instance_id`.
  - `invite`: a guest limited to one room, with the identity `<inviter>/<name>`. Named agents can't contain `/`, so an invite can never impersonate one. A single agent name can't hold tokens in more than one scope.
- Shared helpers in `deps.py` that every route should use:
  - `Caller`: resolves the bearer token to a `Principal`, rejecting tokens that are revoked or expired.
  - `require_scope`: rejects a token whose scope isn't allowed on the route.
  - `assert_identity`: rejects a body `from`, `created_by` or `agent` that differs from the token.
  - `require_room` / `require_participant`: 404 if the room is missing, 403 if the caller hasn't joined (or holds an invite for another room). `require_participant` also refreshes `last_seen_at`.
  - `require_writable`: 409 if the room is archived.
  - `db.audit(...)`: call it inside the same transaction as every write.
- `db.py`: the schema follows gist §12 plus `tokens` and `audit` tables. It is created with `create table if not exists` on startup, and there is no migration tool yet.
- `models.py`: the optional typed-message fields (`confidence`, `reply_requested`, `severity`, `based_on_messages`) are stored in `messages.payload_json`. To add one, list it in `PAYLOAD_FIELDS` and add it to both models.
- Tests use FastAPI's `TestClient` against a temporary database. The fixtures in `tests/conftest.py` (`make_agent(name, scope)`, `boostie`, `missy`, `agentd1`, `room_id`) issue real tokens. To test expiry, set `expires_at` in the past directly in SQL.
- MVP access rules: any `agent`-scope token can join any room, and an invite can join only its own room. Every other room route requires being a participant. Per-room permissions come later.

## agentd commands (run from `agents/`)

```bash
uv sync && uv run pytest -q               # about 10s; tests start real fake-worker subprocesses
uv run pytest tests/test_sessions.py::test_stop_escalates_to_kill
uv run ruff check . && uv run ruff format .

export AGENTD_CONFIG=agentd.example.yaml AGENTD_DATA_DIR=.data   # all keys: config.Settings
uv run agentd config                      # print the effective config (YAML plus AGENTD_* env)
uv run agentd token create <caller>       # caller bearer token (agentd has its own token DB)
uv run agentd serve [--port 8765]
```

Milestone 1 check: `curl -XPOST localhost:8765/v1/sessions -H "Authorization: Bearer $T" -H 'content-type: application/json' -d '{"task":"say hello","profile":"workspace_coder","worker_type":"fake"}'`, then `curl -N localhost:8765/v1/sessions/<id>/events -H "Authorization: Bearer $T"`. The first word of the task picks the fake worker's mode (`interactive`, `hang`, `crash`, `stubborn`, …); see `workers/fake.py`.

## agentd code layout

- `supervisor.py` is the core. Each live session has one `_run` task that owns the worker process: it reads stdout and stderr, then waits for exit. **`_run` is the only place a session that started becomes terminal.** `stop()`, the ready watchdog and the cleanup loop only record a `stop_status` and signal the process; `_run` then applies `final_seen` → completed, else `stop_status`, else failed. `_set_terminal` is also called directly for sessions that never got a live process: those whose worker failed to launch, and those `recover()` finds left over from a previous gateway process.
- Everything runs on the event loop thread with one shared sqlite3 connection, so **all routes are `async def`**. A sync route would run in a threadpool and share the connection across threads.
- `protocol.py` defines the stdin and stdout protocol. Workers may emit only `progress`, `artifact`, `needs_input`, `final` and `error`. The gateway reserves `status`, `log`, `message`, `protocol_error`, `room_error` and `log_truncated`.
- `events.py` (`EventStore`) writes every event to SQLite and to `events.jsonl`, then wakes SSE readers. The SSE loop relies on there being no `await` between an empty `since()` and `wait()`; keep it that way.
- `backends/process.py`: each worker runs as a subprocess in its own process group (`killpg`), with stop message → SIGTERM → SIGKILL spaced by `stop_grace_seconds`. This backend gives **no isolation**: profile fields other than the runtime limit and the workspace allowlist are only advisory until the Docker backend exists. A worker gets only the env vars in `env_allowlist` plus `AGENTD_*` and, when invited to a room, `ROOMSD_*`.
- Worker types are pure config (`worker_types: {name: {command, env}}`). The built-in set has only `fake`. A real agent CLI needs an adapter that speaks the protocol (milestone 2).
- `rooms_client.py`: the registry heartbeat runs every ttl/3 and also immediately when `Supervisor.capacity_changed` fires. The instance deregisters on shutdown. A room close-out joins, posts, then revokes the invite token.
- Sessions are visible only to the caller that requested them; anyone else gets 404. The room invite token is never written to disk (it's redacted in `input.json`).
- Tests (`tests/helpers.py`): `spawn`, `wait_status`, `events` (the JSON form of `/events`) and `wait_event`. The fixtures set short timeouts (ready 3s, grace 1s, cleanup every 0.2s) and `max_sessions=2`. The roomsd integration has no automated test yet; it was checked by hand against a live roomsd.

## How the two services relate

They solve different problems. Dependencies run one way only: agentd and its workers are clients of roomsd, and roomsd never calls agentd (see "Integration" below).

- **roomsd**: peer collaboration. Independent agents (Boostie, Missy, Claude, Odin, …) that already exist share durable rooms. It **never** spawns or schedules agents, makes model calls, runs tools for agents, or decides who is right. A room holds a chat log of typed messages, notes (blackboard state), tasks with lease-based claims, an artifact index, and a decision log.
- **agentd**: worker orchestration. An always-on gateway that lazily spawns ephemeral helper workers (process at first, Docker later). Callers talk to a *session*, not a shell process. It owns lifecycle: spawn, message, SSE events, status, stop, idle and hard-timeout cleanup.

## roomsd: key invariants

- HTTP API under `/v1/rooms/...`. v1 transport is **polling** (`GET .../messages?after_id=n`), so message IDs must be monotonically increasing integers. SSE comes in v1.1.
- Message types are a small fixed set: message, proposal, objection, question, answer, finding, status, decision_request, decision, artifact, task_update, handoff.
- **Sender identity comes from the auth token**, never from the request body's `from`. Reject a mismatch.
- The message log is append-only. Audit joins, writes, claims, and decisions.
- Task claims are leases: one active claim at a time, and a claim can be replaced once it expires.
- Artifacts are copied or registered into `/var/lib/roomsd/rooms/<id>/artifacts/`. Never serve arbitrary filesystem paths. Enforce max message and artifact sizes.
- Roles (chair, implementer, reviewer, …) are advisory and not enforced in v1.
- MVP order: SQLite, create room, join, post and read messages, put and get notes, bearer auth, then a CLI to tail a room. After that: tasks, artifacts, decisions, SSE, per-room permissions.

## agentd: key invariants

- Profiles are server-side **allowlists** (network, filesystem, mounts, external_actions, max_runtime). Callers pick a profile by name and can never pass raw permissions. A caller-supplied timeout can only *lower* the profile's maximum.
- The worker protocol uses structured JSON-lines events (status, progress, artifact, needs_input, final, log). Never treat raw terminal output as the source of truth. Text-only agents go through an adapter that parses `AGENT_EVENT {json}` lines and wraps everything else as `{"type":"log"}`.
- Session events go to both SQLite and `/var/lib/agentd/sessions/<id>/events.jsonl` (kept for debugging).
- Security: workspace path allowlist, no arbitrary host mounts, env-var allowlist, no inherited secrets. Docker workers run non-root, read-only workspace unless the profile grants write, CPU/memory limits, no privileged mode, no Docker socket.
- Failure handling: mark the session failed with a clear event rather than leaving it looking "still thinking". This covers invalid worker JSON, a worker that never becomes ready, artifact path escapes, a gateway restart with live sessions, and concurrent senders.
- MVP order: create, get, and events endpoints; process backend; JSONL log; one profile (`workspace_coder`); one worker type; idle cleanup. Docker and more profiles come later. Milestone 1 uses a `fake` worker type on `localhost:8765`.

## agentd: scale requirements (from the owner, not in the gist)

Plan for many agentd deployments from the start, even while the MVP stays small. Three dimensions:

- **Many agentd instances** (e.g. one per host). Give each instance a stable `instance_id` from config. Session IDs must be globally unique (ULID-style, like the gist's `agt_01…`), and status and events should say which instance owns a session. Each instance keeps its own SQLite database and `/var/lib/agentd`, so don't assume shared storage. Callers find instances through the roomsd registry (below).
- **Many worker types** (Codex, Claude, OpenClaw, wrapped scripts). Put workers behind a pluggable backend interface selected by `worker_type`. Keep this separate from profiles, which control permissions, and from the runner, which is a process or Docker. "One worker type first" is only MVP ordering; don't hard-code anything specific to one worker type in the gateway.
- **Many concurrent sessions per instance.** Enforce per-instance limits on how many sessions run at once and on resources, and reject or queue requests over the limit with a clear status. One slow or chatty session must never block another's event streaming or cleanup.

## lobbyd: identity and directory (see `design/multi-server.md`)

lobbyd is built. **roomsd and agentd don't use it yet**: they still have their own token tables, and the agentd registry still lives in roomsd. Migrating them is the next step, and the "Changes to existing code" section of the design doc lists what it involves. Read that doc before changing auth or the registry.

Commands (run from `lobby/`; default port 8767):

```bash
uv sync && uv run pytest -q
export LOBBYD_DATA_DIR=.data LOBBYD_ISSUER=http://127.0.0.1:8767 LOBBYD_DOMAIN=local
uv run lobbyd key create <name> [--scope agent|agentd|roomsd]   # long-lived API key
uv run lobbyd signing-key list|rotate|retire <kid>
uv run lobbyd serve
```

Layout and rules:
- `POST /v1/token {audience}`, called with an API key, returns an EdDSA JWT. Its claims are `iss`=`LOBBYD_ISSUER`, `sub`=`name@domain`, `aud`=the target service's base URL, `scope`, and `exp` (default 15 minutes). lobbyd's own endpoints accept the **API key directly**, not access tokens.
- `signing.py`: the newest unretired key signs, and every unretired key is published at `/.well-known/jwks.json`. Rotating is `rotate` now, then `retire` the old key after at least one token lifetime. Private keys live in the SQLite file, so `init_db` makes the data dir `0700`.
- `verify.py` (`TokenVerifier`) is **meant to be copied into roomsd and agentd**; it depends only on PyJWT and httpx. It checks `iss`, `aud`, `exp` and that `sub` belongs to the issuer's domain. It caches the JWKS and refetches on an unknown `kid`, at most once per `min_refresh_seconds`.
- `routes/directory.py`: roomsd servers (`/v1/servers/roomsd/{server_id}`) and agentd instances (`/v1/registry/agentd/{instance_id}`) are heartbeat leases, and only the key with that name can write its entry. Listed rooms (`/v1/rooms`) can be written only by a registered, live roomsd, only for URLs under its own `base_url/v1/rooms/`. They are hidden while that server's lease has lapsed.

## Integration: roomsd as the agentd registry (owner decision, not in either gist)

roomsd acts as the registry of agentd instances. Orchestrating agents (Missy, Claude, OpenClaw) bring workers into rooms. The flow, with the roomsd side implemented:

1. **Register.** The agentd instance, using its `agentd`-scope token, calls `PUT /v1/registry/agentd/{instance_id}` with `base_url`, `worker_types`, `profiles`, `max_sessions`, `active_sessions`, and optional `metadata` and `ttl_seconds` (default 60, max 600). Sending the same PUT again is the heartbeat; send it about every ttl/3. Entries past `expires_at` drop out of lookups. `DELETE` the entry on a clean shutdown.
2. **Create a room.** The orchestrator calls `POST /v1/rooms`.
3. **Pick an instance.** The orchestrator calls `GET /v1/registry/agentd?worker_type=…&profile=…&has_capacity=true`. Results come back sorted with the most spare capacity first.
4. **Invite.** The orchestrator calls `POST /v1/rooms/{id}/invites` with `{name, role, ttl_seconds}` (default 1h, max 24h) and gets back a token whose identity is `<orchestrator>/<name>`. Name the worker after where it runs, e.g. `agentd-host1.codex`. The orchestrator then calls the instance's `POST /v1/sessions` with the roomsd URL, `room_id` and invite token. The spawn request carries `room: {room_id, token, url?}`, and `url` defaults to agentd's `roomsd_url`.
5. **Join.** agentd hands the token to the worker through its allowlisted environment, never as a profile permission. The worker calls `POST /v1/rooms/{id}/participants`, and its role is fixed to the role on the invite. Its first message should be a `status` message naming `instance_id` and `session_id`.
6. **Finish.** When the session ends for any reason, agentd (`Supervisor._close_out_room`) posts a closing `status` or `handoff` message and then calls `POST /v1/auth/revoke` with the worker's token, so the token dies with the session. The inviter or the room creator can also revoke it with `DELETE /v1/rooms/{id}/invites/{invite_id}`. `GET /v1/auth/whoami` returns any token's identity, scope, room and expiry.

Constraints this keeps:

- roomsd still never spawns, schedules or calls agents. The orchestrator does the spawning, through agentd.
- Sender identity still comes from the token. A worker can't post as its orchestrator or as another worker.
- agentd never needs room-admin rights. It only passes along the invite token it was given.
- Invitees can't create rooms, issue further invites or read the registry. Registry lookups are for `agent`-scope tokens only.
