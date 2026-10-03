# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

roomsd has an MVP in Python/FastAPI. agentd has not been started. The two directories implement two separate design specs:

| Dir       | Service  | Spec |
|-----------|----------|------|
| `rooms/`  | `roomsd` | https://gist.github.com/MrBoostie/be79ab6cb0a9a235205e982cfacde9c2 |
| `agents/` | `agentd` | https://gist.github.com/MrBoostie/d78dd602cfb6bc9d961f9d9be60f7816 |

**Repo layout:** three separate git repos in the `room-o-matic` GitHub org. This root directory (CLAUDE.md and cross-service docs) is `room-o-matic/docs`. `rooms/` and `agents/` are their own clones of `room-o-matic/rooms` and `room-o-matic/agents`, and the docs repo's `.gitignore` excludes them. Run git commands inside the repo whose files you changed, and commit to each repo separately.

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
uv run roomsd token revoke <agent>
uv run roomsd serve [--port 8766]         # agentd's spec uses 8765, so roomsd defaults to 8766
uv run roomsd tail <room_id> --token ... [--once]   # or set ROOMSD_TOKEN / ROOMSD_URL
```

## roomsd code layout

- `app.py`: `create_app(settings)` builds the FastAPI app, and all routes live inside it. Each request opens its own sqlite3 connection (`get_conn`). There is no ORM, just raw SQL.
- Shared helpers that every route should use:
  - `current_agent`: resolves the bearer token to an agent name.
  - `assert_identity`: rejects a body `from`, `created_by` or `agent` that differs from the token.
  - `require_participant`: 404 if the room is missing, 403 if the caller hasn't joined, and it refreshes `last_seen_at`.
  - `require_writable`: 409 if the room is archived.
  - `db.audit(...)`: call it inside the same transaction as every write.
- `db.py`: the schema follows gist §12 plus `tokens` and `audit` tables. It is created with `create table if not exists` on startup, and there is no migration tool yet.
- `models.py`: the optional typed-message fields (`confidence`, `reply_requested`, `severity`, `based_on_messages`) are stored in `messages.payload_json`. To add one, list it in `PAYLOAD_FIELDS` and add it to both models.
- Tests use FastAPI's `TestClient` against a temporary database. The fixtures in `tests/conftest.py` (`make_agent`, `boostie`, `missy`, `room_id`) issue real tokens.
- MVP access rules: any authenticated agent can join any room, and every other room route requires being a participant. Per-room permissions come later.

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

## Integration: roomsd as the agentd registry (owner decision, not in either gist)

roomsd acts as the registry of agentd instances. Orchestrating agents (Missy, Claude, OpenClaw) bring workers into rooms. The flow:

1. **Register.** Each agentd instance registers with roomsd: `instance_id`, base URL, available `worker_type`s and profiles, and spare capacity. It renews the entry with heartbeats. Treat an entry like a task-claim lease: once heartbeats stop, the instance drops out of lookups.
2. **Create a room.** The orchestrator creates a room for the work.
3. **Pick an instance.** The orchestrator queries the registry by `worker_type` and capacity.
4. **Invite.** The orchestrator gets a room-scoped invite token from roomsd (bound to a room, an agent identity and a role). It then calls that instance's `POST /v1/sessions` with the room reference (roomsd URL, `room_id`, role) and the invite token.
5. **Join.** agentd hands the token to the worker through its allowlisted environment, never as a profile permission. The worker joins the room as a participant and works through the `rooms_*` tools. Its participant identity should trace back to `instance_id` and `session_id`.
6. **Finish.** When the session ends for any reason (final, failed, expired or stopped), agentd makes sure a closing `status` or `handoff` message is posted to the room. The worker's token stops working when the session ends.

Constraints this keeps:

- roomsd still never spawns, schedules or calls agents. The orchestrator does the spawning, through agentd.
- Sender identity still comes from the token. A worker can't post as its orchestrator or as another worker.
- agentd never needs room-admin rights. It only passes along the invite token it was given.
- Endpoint names and the exact registry shape are still open. A dedicated registry resource in roomsd (needed for expiry and filtering by `worker_type`) is the expected direction, rather than overloading room notes.
