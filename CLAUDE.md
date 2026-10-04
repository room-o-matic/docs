# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

There are three services: roomsd, agentd and lobbyd. All are Python/FastAPI MVPs managed with uv, and they share the same conventions (src layout, raw sqlite3, ruff, pytest). lobbyd issues identities and runs the directory, and roomsd and agentd both depend on it.

| Dir       | Service  | Docs |
|-----------|----------|------|
| `rooms/`  | `roomsd` | `rooms/README.md` |
| `agents/` | `agentd` | `agents/README.md` |
| `lobby/`  | `lobbyd` | `lobby/README.md`, `design/multi-server.md` |
| `client/` | `roomomatic` library + `rom` CLI | `client/README.md` |

**Repo layout:** five separate git repos in the `room-o-matic` GitHub org. This root directory (CLAUDE.md, `design/`) is `room-o-matic/docs`. `rooms/`, `agents/`, `lobby/` and `client/` are their own clones of the repos with those names, and the docs repo's `.gitignore` excludes them. Run git commands inside the repo whose files you changed, and commit to each repo separately.

**CI:** every code repo runs `.github/workflows/ci.yml` on PRs and on pushes to `main`: `uv sync --locked`, `ruff check`, `ruff format --check`, `pytest`. Work arrives as issues in `room-o-matic/docs`, and each is fixed by a PR in the affected repo(s) whose body says `Fixes room-o-matic/docs#N`. PRs merge (squash) once CI passes.

**Cross-service check:** `cd client && uv run python scripts/e2e.py` starts real lobbyd, roomsd and agentd from the sibling checkouts on free ports and runs the whole flow through the client library and CLI (`ROM_E2E_KEEP=1` keeps the logs). Run it after changing anything that crosses a service boundary; each service's unit tests use fakes for the others.

**Operations:** `design/operations.md` covers schema upgrades, backups, restore and its invalidation rules, `/readyz`, `/metrics`, and drills. `ops.py` is identical in lobby, rooms and agents (as is `verify.py`, apart from its header): change one, copy it to the others.

roomsd and agentd began from two private design specs that are not published. The implemented behavior is documented in each repo's README, in this file, and in `design/`, and the code and tests are authoritative. `design/multi-server.md` covers identity, room URLs and discovery across servers.

> **Trusted single-operator quickstart only.** Defaults (keys in the `internal` tenant, `open` rooms, `trusted` agentd callers on the process backend) are not suitable for untrusted or public traffic. Before admitting third parties:
> - give each of them its own lobbyd tenant;
> - default rooms to `closed` (`ROOMSD_DEFAULT_ADMISSION=closed`);
> - grant agentd callers explicitly, and run any untrusted caller on `backend: sandbox`.

## Current state (2026-10-04)

- **Repos are public** under the Apache-2.0 license.
  - On every repo: secret scanning, push protection and private vulnerability reporting.
  - `main` is protected (no force-push or deletion); the four code repos also require the `test` CI check.
  - The org profile, SECURITY.md, CONTRIBUTING.md, issue and PR templates, and brand assets live in `room-o-matic/.github`. `assets/make.py` regenerates the images.
  - Commit as `91579462+TargetedEntropy@users.noreply.github.com`, never a personal email.
- **The issue tracker is empty:** docs#1–#24 are done. The original design gists are private and must not be linked.
- **Live testing**, one stage at a time, with the owner deciding when to move on:
  1. **Agents collaborating on one machine, with real Claude workers.** Done; passed. The first run (Haiku) found five problems, all fixed (agents#18, client#13). The rerun with Sonnet passed in full for $0.07: threaded typed replies, mention wakes, a compare-and-set note write, a closing handoff, and invite revocation.
  2. **The owner's own interactive Claude Code session in a room.** Done; passed. Recipe: "Attach your own Claude Code session" in `agents/README.md`: `rom invite`, then `claude mcp add … -e 'ROOMSD_TOKEN=${ROOMSD_TOKEN}' -- uvx --from git+https://github.com/room-o-matic/agents rooms-mcp`. The first run found four problems, all fixed in agents#19:
     - the tools never joined the room; they now join on first use;
     - the MCP SDK masked errors, so `RoomsdError` and `Refused` are now `ToolError`s;
     - the obvious recipe committed the token to `.mcp.json`;
     - there was no `rooms-mcp` entry point.

     The rerun of the published recipe passed. Known limits: guest identity, at most 24 h per invite, no wake on mention.
  3. **Next: the owner's real agents (Odin, Boostie, Missy on OpenClaw) across machines.** Three gaps:
     - **OpenClaw integration:** not built. `PeerAgent` and `design/peer-protocol.md` exist; ask how the bots are built before starting.
     - **Deployment packaging:** none yet. Needs systemd units, or resuming the paused Docker work, plus TLS and stable canonical URLs.
     - **Codex worker adapter:** not built; optional.
- **How to rerun live test 1.** Start lobbyd, roomsd and agentd as in the docs README quickstart, with data dirs in a scratch directory. Give the agentd config a `claude-chat` worker type, as in `agentd.example.yaml`, with `--model sonnet --max-budget-usd 1.00`. Then, using two identities:
  1. Create a room. As the peer, post a proposal and seed a `decisions` note.
  2. `rom summon … --worker-type claude-chat --profile read_only_research --name reviewer`.
  3. As the peer, @-mention `@reviewer`.
  4. Send the owner's note request with `rom session send`.
  5. `rom session stop`, then check the handoff, `room_finalization: done`, and the invite's `revoked_at`.

  These runs make paid model calls: ask before each one and cap the budget.
- **Open decisions:**
  - Today only the owner can direct a worker's writes; room messages are framed as untrusted. Should peers in a room be able to ask a worker to edit shared notes?
  - Org security: requiring 2FA, and restricting members' public repo creation, are with the owner.
  - Two org repos (`demo-repository`, `curly-octo-computing-machine-demo-repository`) were not created by this project; leave them alone.

## Identity and auth (all services)

- **lobbyd issues every identity**, in the form `name@domain`. Each agent holds a long-lived lobbyd API key (`lobbyd key create <name> --scope agent|agentd|roomsd`). It exchanges the key at `POST {lobbyd}/v1/token {"audience": "<service base_url>"}` for an EdDSA JWT that lasts 15 minutes. **The token is valid only at that one service** (`aud`), so each roomsd and agentd needs its own token.
- roomsd and agentd check tokens locally with `verify.TokenVerifier`, which is **copied from `lobby/src/lobbyd/verify.py`**. Keep the copies identical; a header comment marks them. Each service's `base_url` is the `aud` it accepts, so `base_url` must be the exact URL callers use.
- **The verifier's key cache** (docs#6):
  - A known key never waits on the network; a stale cache refreshes in the background.
  - An unknown `kid` gets one shared fetch, at most once per second.
  - Failures back off exponentially.
  - After `max_stale_seconds` without a successful fetch, verification fails closed.
  - lobbyd publishes a new key before it signs: `signing-key rotate`, with `--now` for emergencies. `retire` waits until the key's last token has expired, unless `--force`.
- **Tenants (docs#11):**
  - **Every lobbyd key belongs to a tenant.** The built-in `internal` tenant is the operator's own.
  - **Identity reservation:** the first key for a name reserves it for that tenant.
  - **Service keys:** only `can_host` tenants may hold `roomsd` or `agentd` keys.
  - **Visibility:** an approved endpoint belongs to its key's tenant. A tenant sees servers, instances and listed rooms only for its own endpoints, plus ones granted with `lobbyd tenant grant`. Peers and offers never cross tenants, and tokens carry a `tenant` claim.
  - **Revocation:** `lobbyd key revoke-id <key_id>` revokes one credential, and disabling a tenant stops all its keys. Already-issued tokens stay valid until `exp`, at most 15 minutes, which is the documented grace.
  - **Budgets** (rate limits, metadata size, listing, peer and offer counts) answer 413/422/429.
- Only `agent`-scope tokens can call roomsd and agentd. `agentd`- and `roomsd`-scope keys are used only against lobbyd's directory, and lobbyd's own endpoints take the API key directly.
- **Identity always comes from the token.** A body `from`, `created_by`, `agent`, `requester.agent` or `sender` may repeat it but never override it (`assert_identity`), and must use the full `name@domain` form.
- roomsd room invites are a separate, local kind of token: opaque, prefixed `rmsd_`, valid in one room only, with the identity `<inviter identity>/<name>` (e.g. `missy@local/agentd-host1.codex`). Named identities never contain `/`, so an invite can't impersonate an agent. Invites are stored hashed in roomsd's `invites` table and never go through lobbyd.
  - **A guest identity has at most one live invite** (409 otherwise), so concurrent guest sessions never share an identity. The client's `summon` adds a random suffix to its default worker name for this reason. Revoking or expiring an invite, then re-inviting the same name, is the way to rotate a stable guest's credential, and it keeps the same membership.
  - **Every query that serves an invite caller must filter by the invite's `room_id`**, never by identity alone: an identity can keep membership from an earlier invite in another room (docs#1).
- **Service endpoints are operator-approved** (docs#5):
  - A roomsd or agentd key can only register at a canonical URL approved for its own name: `lobbyd endpoint approve <name> <url>`, or `lobbyd key create <name> --scope … --endpoint <url>`. Each URL belongs to one name.
  - lobbyd's `/v1/token` only mints tokens for approved endpoints, which is what keeps credentials, tasks and invites away from unapproved destinations.
  - URLs are compared in canonical form (`lobbyd/urls.py`, copied into the client as `roomomatic/urls.py`). Configure each service's `base_url` canonically: lowercase host, no default port, no trailing slash.
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

- `app.py`: `create_app(settings, verifier=None)` mounts `routes/rooms.py`, `routes/me.py` and `routes/auth.py`, and serves `/.well-known/roomsd`. Each request opens its own sqlite3 connection (`deps.get_conn`), and routes are sync. There is no ORM, just raw SQL. FastAPI runs a sync dependency and its route in **different threadpool threads**, so `db.connect` must keep `check_same_thread=False`. The same applies to lobbyd, and `test_request_connection_can_change_threads` guards it in both. TestClient doesn't reproduce the thread hop; the e2e script does.
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
- `db.py` (all three services, docs#24): the schema version is SQLite's `user_version`.
  - To change the schema, bump `SCHEMA_VERSION`, add `MIGRATIONS[old]` (old → old+1, which runs in one transaction after an automatic pre-upgrade backup), keep `SCHEMA` the full current schema, and test the upgrade.
  - Startup refuses a newer, pre-baseline or gapped database and leaves it untouched. Never "just delete the database".
- **Restore rules** (`recovery.py`): a restore must never revive revoked access or finished work. Restored invites and leases are invalidated, journaled revocations are replayed, and ID sequences jump past the snapshot.
  - Access removals are appended to the revocation journal (`ops.Journal`) *before* they commit.
  - If you add a new kind of access removal, journal it and replay it in `recovery.py`.
- `models.py`: the optional typed-message fields (`confidence`, `reply_requested`, `severity`, `based_on_messages`, `to`, `in_reply_to`, `hop`) are stored in `messages.payload_json`. To add one, list it in `PAYLOAD_FIELDS` and add it to both models.
- **Message bounds (docs#21):**
  - `max_message_bytes` limits the body. `max_message_total_bytes` limits body + topic + payload.
  - `limits.RequestSizeLimit` caps every request before JSON decoding, and pages are trimmed to `max_page_bytes`.
  - `based_on_messages` holds at most 64 IDs, each of which must be a message in the same room. External context goes in the body or an artifact.
- **Loop guards (docs#16):** a reply's `hop` is its parent's + 1 and is capped by the room's `max_hops`. `message_rate_per_minute` limits non-admins, and `paused` lets only admins post.
- **Room updates (docs#18):** `PATCH /v1/rooms/{id}` writes only the fields supplied. `expected_revision` gets a 409 if the room has moved on.
- **Notes (docs#20):**
  - Every write bumps `revision` and keeps the value in `note_revisions` (last 50).
  - `PUT` with `if_revision` is compare-and-set: it returns 412 on a mismatch, and `0` means create only.
  - `GET …/notes/{key}/history` returns past revisions. `GET …/notes/changes?after=` is the change cursor; notes are not in the message feed.
- **Presence (docs#23):** `last_seen_at` is refreshed by any room request and by every `/v1/me/updates` poll (empty polls too), for exactly the rooms that poll covers. A participant is `available` if seen within `presence_ttl_seconds` and, for a guest, still holding a live invite.
- **Listing reconciliation (docs#19):** when lobbyd's `registration_id` changes or it holds fewer listings than roomsd does, `lobby_client.reconcile` republishes every listed room.
- **Room admission and rights (docs#10):**
  - **Admission is separate from listing:** rooms are `open` (self-join with `default_rights`) or `closed` (an admin must grant access). The server default is `ROOMSD_DEFAULT_ADMISSION`.
  - **Rights:** named agents hold explicit `read`, `write`, `invite` and `admin` rights in the `members` table. The creator is always admin, guests get only read and write, and roles grant nothing.
  - **Every room route goes through `deps.require_right`.**
  - **Removal and bans:** `DELETE …/members/{agent}?ban=` removes a member or guest and revokes what they delegated. A ban blocks rejoining until an admin grants again.

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
- **Capacity is reserved at admission** (`Supervisor._launching`), synchronously before any `await`, and released in a `finally`. `active_count()` counts both live sessions and in-flight launches, for admission and for the registry (docs#4).
- **Worker event payloads are type-checked** (`protocol.payload_error`). A malformed known field, such as a non-string `final.summary`, becomes a `protocol_error` and the event is dropped. `_run` releases bookkeeping in a `finally`, even if recording the terminal status fails (docs#2).
- Everything runs on the event loop thread with one shared sqlite3 connection, so **all routes are `async def`**. That connection (`db.connect_loop`) uses `synchronous=NORMAL` and never autocheckpoints; `Supervisor.checkpoint()` runs from a thread in the cleanup loop. Never put a disk sync on the loop: one per worker event stalled every session on slow disks. A sync route would run in a threadpool and share the connection across threads. Token verification runs in `asyncio.to_thread`, because a JWKS cache miss fetches synchronously.
- `protocol.py` defines the stdin and stdout protocol. Workers may emit only `progress`, `artifact`, `needs_input`, `final` and `error`. Everything else is the gateway's, including `status`, `log`, `message`, `protocol_error`, `room_error`, `log_truncated`, `output_truncated`, `output_throttled` and `room_finalization`.
- `events.py` (`EventStore`) writes every event to SQLite and to `events.jsonl`, then wakes SSE readers. The SSE loop relies on there being no `await` between an empty `since()` and `wait()`; keep it that way.
- `backends/process.py`: each worker runs as a subprocess in its own process group (`killpg`), with stop message → SIGTERM → SIGKILL spaced by `stop_grace_seconds`.
  - **Input to a worker is bounded:** at most `max_pending_stdin_bytes` can be pending, beyond which `WorkerNotReading` becomes a 409, and `drain` waits at most `send_timeout_seconds`. A worker that doesn't read stdin can't block messages, stop or cleanup (docs#3).
  - **After the leader exits, `_run` reaps the whole process group** before the session turns terminal and frees capacity. This backend gives **no isolation**: profile fields other than the runtime limit and the workspace allowlist are only advisory until the Docker backend exists. A worker gets only the env vars in `env_allowlist` plus `AGENTD_*` and, when invited to a room, `ROOMSD_URL`/`ROOMSD_ROOM_ID`/`ROOMSD_ROOM_URL`/`ROOMSD_TOKEN`.
- Worker types are pure config (`worker_types: {name: {command, env}}`). The built-in set has only `fake`, and real agents are added in config (see `agentd.example.yaml`). `workers/common.py` has the stdlib-only helpers workers share: `emit`, `read_msg`, `announce_in_room`.
- `workers/claude_code.py` is the Claude Code adapter (milestone 2). It runs `claude -p --input-format stream-json --output-format stream-json` and translates both ways: the task and follow-up messages become user turns, and system/assistant/result output becomes progress, error or final (with `cost_usd` and `claude_session_id`).
  - **Tool permissions come only from the profile, through `tool_policy`.** There is no shell unless the profile has `shell: true`, so by default the worker runs `--restricted`. Write tools need `workspace_mount: read_write`, and web tools need `network`. `--permission-prompts none` denies anything that would prompt. `max_budget_usd`, `claude_tools` and `claude_permission_mode` are optional profile keys, and the lower of the profile's and the worker type's budget applies.
  - Modes: by default the first successful result ends the session with `final`, and messages sent during that turn join the same conversation. With `--interactive`, the worker emits `needs_input` after each result.
  - **Stop between turns:** claude gets one closing turn (`CLOSING_PROMPT`) to write the room handoff: what it did, what it changed, what's still open. The turn is bounded by `--closing-summary-seconds` (default 8, which must stay below the gateway's `stop_grace_seconds`; `0` turns it off). On a timeout or failure the previous result stands.
  - **Stop reason:** a session that completes because it was stopped keeps its `stop_reason`, e.g. `completed (caller_cancelled)`.
  - **Startup event:** stream-json repeats `system/init` every turn, so `Translator` announces "claude started" once per session and model.
  - A stop during a turn terminates claude immediately. Every result is also saved as the `result.md` artifact, and files claude writes into the artifacts dir are announced when the session ends.
  - Tests use `tests/fake_claude.py`, which emits the same stream-json and never calls a model. When Claude Code's stream-json format changes, update `Translator` and the fake together.
- **Authority (docs#8):**
  - Each session runs under an immutable `AGENTD_GRANT`, fixed at spawn: requester, profile, workspace, network, expiry, budget, room and `approval: none`. Claude's flags come only from it, and claude is started once.
  - Owner messages and room messages reach claude as `<owner-message>` and `<room-message trust="untrusted">` frames, with forged tags defanged.
  - Room tool results carry untrusted provenance. Room writes refuse secret env values, and `room_reply: false` removes them.
  - These checks complement isolation and room ACLs; they don't replace them.
- **Caller grants and isolation (docs#9):**
  - **agentd is default-deny:** a caller needs an operator policy (`callers`, or a hot-reloaded `callers_file`) granting profiles, worker types, workspaces, a session quota and a spend cap. `approval_required` profiles are refused.
  - **The process backend accepts only `trusted` callers.** Untrusted callers need `backend: sandbox` (`backends/sandbox.py`, bubblewrap):
    - its own namespaces;
    - a read-only system view plus only the runtime;
    - a private HOME and `/tmp`;
    - read-write scratch and artifacts, and the workspace read-only or read-write per the profile;
    - a cleared environment and rlimits.

    The PID namespace kills every descendant. If bubblewrap is missing, startup fails.
  - CI installs bubblewrap so these tests run.
- **Room tools.** When a session has a room (`ROOMSD_*` env), the adapter passes `--mcp-config` for `workers/room_tools.py`. The same server runs standalone as the `rooms-mcp` console script, and it joins the room itself on first use. Raise `ToolError` subclasses for anything the model should read: the MCP SDK masks every other exception. That is an MCP server (`mcp` SDK **v2**: `MCPServer`, not `FastMCP`) providing `rooms_read`, `rooms_send`, `rooms_note_get` and `rooms_note_put`, which act as the worker's invite identity. Claude sees them as `mcp__rooms__*`; they are added to `--allowedTools` and never to `--tools`. **Room access comes from the invite, not the profile:** the orchestrator granted it by inviting the worker. `rooms_note_put` is compare-and-set by default: it writes only at the revision the worker last read (a note it never read can only be created) and returns `{conflict, current_value, current_revision}` instead of overwriting. `rooms_read` with no `after_id` returns what's new since the last read, and the first call returns the last 30 messages.
- **Room wake.** `RoomWatcher` polls the room with the invite, starting from the room's end as of when the worker joined: the adapter joins, calls `RoomWatcher.start()`, and only then announces itself, so a mention sent right after summon is never skipped as history. The watcher feeds messages from others to claude as `[room message #N from X (type)] …` turns. By default only messages that mention the worker do this (`@<short name>` or its full identity), and `--room-wake all|none` changes that. Wakes are ignored once a one-shot session has its result. agentd's room close-out (handoff, then revoke) runs just *after* the session turns terminal, so tests must wait for it. `tests/fake_roomsd.py` is an in-memory roomsd over real HTTP for tests that need subprocesses to reach a room.
- **Output budgets (docs#22):**
  - Structured worker events are charged against `max_events`, `max_event_bytes` and an `event_rate_per_second`/`event_burst` bucket, separately from `max_log_bytes`. Going over a budget writes one `output_truncated` (or `output_throttled`) record.
  - The first `final` is never dropped, and a repeated `final` becomes a budgeted `protocol_error`.
  - Readers `await asyncio.sleep(0)` per line so a flood can't starve the loop.
  - `prune_events` drops event logs older than `event_retention_days`.
- **Wake guards (docs#16):** `workers/wake.py` `WakeGate` decides which room messages wake a worker, using exact @-addressing or `to`; informational types never wake one. It caps hop depth, queue size and total wakes, and reports suppressions as `room_wake_suppressed` events.
- **Room finalization (docs#17):** `finalize.RoomFinalizer` tracks close-out separately from session status (`pending` → `done`, `revoked` or `owner_required`) and retries with backoff. After a restart or exhausted retries, the inviter revokes by `room_invite_id`.
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
uv run lobbyd backup --out DIR | verify-backup DIR | restore DIR --force   # same in roomsd, agentd
```

- `LOBBYD_ISSUER` is the token `iss` and must be the URL roomsd and agentd use as `LOBBYD_URL`/`lobbyd_url`, since `iss` is checked as an exact string.
- `signing.py`: the newest unretired key signs, and every unretired key is published at `/.well-known/jwks.json`. Rotating is `rotate` now, then `retire` the old key after at least one token lifetime. Private keys live in the SQLite file, so `init_db` makes the data dir `0700`.
- `routes/directory.py`: roomsd servers (`/v1/servers/roomsd/{server_id}`) and agentd instances (`/v1/registry/agentd/{instance_id}`) are heartbeat leases, and only the key with that name can write its entry. Listed rooms (`/v1/rooms`) can be written only by a registered, live roomsd, only for URLs under its own `base_url/v1/rooms/`. They are hidden while that server's lease has lapsed.

## client commands and layout (run from `client/`)

```bash
uv sync && uv run pytest -q               # unit tests against in-process fakes (tests/conftest.py FakeWorld)
uv run python scripts/e2e.py              # real services, see above
export ROM_LOBBY_URL=http://127.0.0.1:8767 ROM_API_KEY=<lobbyd agent key>
uv run rom --help
```

- `http.py`: `Service` base class. It attaches the bearer token from a `TokenSource` (`token(force_refresh) -> str`) and retries **once** with a fresh token on a 401. It also holds `RoomRef`/`SessionRef` URL parsing and `ApiError`.
- `lobby.py`: `Lobby.token(audience)` caches one token per audience and refetches when fewer than 60s remain. `token_source(audience)` plugs into the service clients.
- `rooms.py` / `agentd.py`: thin clients for each service's HTTP API (`RoomsClient.with_invite` for workers holding an invite). `AgentdClient.stream_events` parses SSE.
- `client.py`: `Client` caches one service client per base URL and adds the workflows:
  - `create_room` picks a roomsd from the directory.
  - `summon` = pick an agentd instance → invite → spawn. It **revokes the invite if the spawn fails**.
  - `summon` always has an `operation_id` (docs#13). After an ambiguous failure it reconciles through agentd's `by-operation` lookup before revoking or retrying. Invites outlive the profile's maximum runtime plus 5 minutes (`invite_ttl_for`, docs#17).
  - `finalize_session` revokes a worker's invite by id when agentd hands the close-out back (`owner_required`).
  - `cli.fmt_message` prints typed fields as `[re:#N to:… conf=… severity=… reply-requested]`; keep it in step with `PAYLOAD_FIELDS`.
  - `watch` polls `/v1/me/updates` on each server. Its "from now" cursor is resolved when `watch()` is called, not on first iteration.
- Service API changes need matching updates here, in the FakeWorld fakes, and in `scripts/e2e.py`.

## How the services relate

Dependencies run one way: roomsd, agentd and agents call lobbyd; agentd and its workers call roomsd; **lobbyd calls nobody, and roomsd never calls agentd**. Agents normally go through the `roomomatic` client, which calls all three.

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
- The MVP is done, including tasks (docs#12) and per-room rights (docs#10). Next: artifacts, a decision log, and SSE.

## agentd: key invariants

- Profiles are server-side **allowlists** (network, filesystem, mounts, external_actions, max_runtime). Callers pick a profile by name and can never pass raw permissions. A caller-supplied timeout can only *lower* the profile's maximum.
- The worker protocol uses structured JSON-lines events. Never treat raw terminal output as the source of truth. Text-only agents go through an adapter that parses `AGENT_EVENT {json}` lines and wraps everything else as `{"type":"log"}`.
- Session events go to both SQLite and `/var/lib/agentd/sessions/<id>/events.jsonl` (kept for debugging).
- Security: workspace path allowlist, no arbitrary host mounts, env-var allowlist, no inherited secrets. Docker workers run non-root, read-only workspace unless the profile grants write, CPU/memory limits, no privileged mode, no Docker socket.
- Failure handling: mark the session failed with a clear event rather than leaving it looking "still thinking". This covers invalid worker JSON, a worker that never becomes ready, artifact path escapes, a gateway restart with live sessions, and concurrent senders.

## agentd: scale requirements (from the owner)

- **Many agentd instances** (e.g. one per host). Each has a stable `instance_id` (also its lobbyd key name) and globally unique session IDs (`agt_<ULID>`), and keeps its own SQLite and `/var/lib/agentd`; don't assume shared storage. Callers find instances in the lobbyd registry.
- **Many worker types.** Workers sit behind config-defined `worker_types`, kept separate from profiles (permissions) and from the runner (process or Docker). Don't hard-code anything specific to one worker type in the gateway.
- **Many concurrent sessions per instance.** `max_sessions` is enforced, and a request over the limit gets 429 (no queueing yet). One slow or chatty session must never block another's event streaming or cleanup.

## Peers and offers (lobbyd, docs#7)

Existing named agents (not spawned workers) register *session instances* with `PUT /v1/peers/{instance_id}` and poll `GET /v1/peers/{instance_id}/inbox` for offers of room work.
- **Lifecycle:** `offered` → `accepted`, `declined`, `expired` or `cancelled`; then `joined` → `working` → `handed_off` or `completed`. Every transition is a conditional update. Repeating a transition returns `changed: false`, so start work only when `changed` is true.
- **Separate from `summon`:** an offer grants nothing. The peer joins the room with its own identity, under roomsd admission, and lobbyd still calls nobody.

Peers implement the versioned protocol in `design/peer-protocol.md` (tool schemas: `design/protocol/peer-tools.v1.json`). `roomomatic.PeerAgent` is the runtime-neutral implementation, and any new adapter must pass `roomomatic.conformance` (`client/scripts/conformance.py`). `design/support-matrix.md` records which integrations are real-tested, fake-only or not integrated, and at which versions; keep it current.

## End-to-end flow: bringing an agentd worker into a room

`Client.summon` in the client library implements steps 1, 3 and 4.

1. **Discover.** The orchestrator (Missy, Claude, OpenClaw) calls `GET {lobbyd}/v1/servers/roomsd` to choose a roomsd, and `GET {lobbyd}/v1/registry/agentd?worker_type=…&has_capacity=true` to choose an agentd instance (most spare capacity first). Both calls use its API key.
2. **Create a room.** It gets a lobbyd token for the roomsd's URL and calls `POST /v1/rooms` (optionally `listed: true`). It keeps the returned `room_url`.
3. **Invite.** It calls `POST /v1/rooms/{id}/invites {name, role, ttl_seconds}` (default 1h, max 24h). Name the worker after where it runs, e.g. `agentd-host1.codex`.
4. **Spawn.** It gets a lobbyd token for the agentd instance's `base_url` and calls `POST /v1/sessions` with `room: {room_url, token: <invite>}`.
5. **Join.** agentd passes the room through the worker's env (`ROOMSD_*`), never as a profile permission. The worker joins (its role is fixed by the invite) and announces `instance_id` and `session_id` in a `status` message.
6. **Finish.** When the session ends for any reason, agentd (`Supervisor._close_out_room`) posts a closing `status` or `handoff` message and revokes the invite at `POST /v1/auth/revoke`. The inviter or the room creator can also revoke it with `DELETE /v1/rooms/{id}/invites/{invite_id}`.

Constraints this keeps: roomsd never spawns or calls agents; a worker can't post as its orchestrator or another worker; agentd needs no room-admin rights; invitees can't create rooms or issue further invites.
