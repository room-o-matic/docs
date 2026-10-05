# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

There are four services (lobbyd, roomsd, agentd, dispatchd) and a client library with a CLI. All are Python, managed with uv, and share the same conventions (src layout, raw sqlite3, ruff, pytest); the services are FastAPI. lobbyd issues identities and runs the directory, and everything else depends on it.

| Dir       | Service  | Docs |
|-----------|----------|------|
| `rooms/`  | `roomsd` | `rooms/README.md` |
| `agents/` | `agentd` | `agents/README.md` |
| `lobby/`  | `lobbyd` | `lobby/README.md`, `design/multi-server.md` |
| `client/` | `roomomatic` library + `rom` CLI | `client/README.md` |
| `dispatch/` | `dispatchd` | `dispatch/README.md`, `design/dispatch.md` |

**Repo layout:** six separate git repos in the `room-o-matic` GitHub org (plus `.github` for the org profile). This root directory (CLAUDE.md, `design/`) is `room-o-matic/docs`. `rooms/`, `agents/`, `lobby/`, `client/` and `dispatch/` are their own clones of the repos with those names, and the docs repo's `.gitignore` excludes them. Run git commands inside the repo whose files you changed, and commit to each repo separately.

**CI:** every code repo runs `.github/workflows/ci.yml` on PRs and on pushes to `main`: `uv sync --locked`, `ruff check`, `ruff format --check`, `pytest`. Work arrives as issues in `room-o-matic/docs`, and each is fixed by a PR in the affected repo(s) whose body says `Fixes room-o-matic/docs#N`. PRs merge (squash) once CI passes.

**Cross-service check:** `cd client && uv run python scripts/e2e.py` starts real lobbyd, roomsd and agentd from the sibling checkouts on free ports and runs the whole flow through the client library and CLI (`ROM_E2E_KEEP=1` keeps the logs). Run it after changing anything that crosses a service boundary; each service's unit tests use fakes for the others.

**Operations:** `design/operations.md` covers schema upgrades, backups, restore and its invalidation rules, `/readyz`, `/metrics`, and drills. `ops.py` is identical in lobby, rooms, agents and dispatch (as is `verify.py`, apart from its header): change one, copy it to the others.

roomsd and agentd began from two private design specs that are not published. The implemented behavior is documented in each repo's README, in this file, and in `design/`, and the code and tests are authoritative. `design/multi-server.md` covers identity, room URLs and discovery across servers.

> **Trusted single-operator quickstart only.** Defaults (keys in the `internal` tenant, `open` rooms, `trusted` agentd callers on the process backend) are not suitable for untrusted or public traffic. Before admitting third parties:
> - give each of them its own lobbyd tenant;
> - default rooms to `closed` (`ROOMSD_DEFAULT_ADMISSION=closed`);
> - grant agentd callers explicitly, and run any untrusted caller on `backend: sandbox`.

## Current state (2026-10-04)

- **Repos:** docs, lobby, rooms, agents, client, dispatch and `.github` are all public under Apache-2.0.
  - **On every repo:** secret scanning, push protection, private vulnerability reporting, and auto-delete of merged branches.
  - **`main` is protected:** no force-push or deletion anywhere, and the five code repos also require the `test` CI check.
  - **`room-o-matic/.github`** holds the org profile, SECURITY.md, CONTRIBUTING.md, templates and brand assets. `assets/make.py` regenerates the images, including one social preview per repo.
  - **Commit as** `91579462+TargetedEntropy@users.noreply.github.com`, never a personal email.
- **Issue tracker:** empty; docs#1–#24 are done. The original design gists are private and must not be linked.
- **Worker adapters, all live-verified on this machine:**

  | adapter | module | verified with | PRs |
  |---|---|---|---|
  | Claude Code | `workers/claude_code.py` | CLI 2.1.288, Sonnet and Haiku | agents#18, #19 |
  | Codex | `workers/codex.py` | CLI 0.158, `gpt-6-sol` (ChatGPT login) | agents#20, #21 |
  | Ollama | `workers/ollama.py` | 0.10.1, `qwen2.5:7b-instruct-q4_K_M`, 6 GB GTX 1660 SUPER (free) | agents#22–#24, #26 |

  Codex and Ollama share the `TurnAdapter` session loop (`workers/turns.py`).
- **Live tests passed:**
  1. **Agents on one machine:** a Claude worker reviewed, answered a mention, wrote a note with compare-and-set, and handed off.
  2. **The owner's own Claude Code session in a room:** the recipe is in `agents/README.md`, "Attach your own Claude Code session". Limits: guest identity, invites last at most 24 h, no wake on mention.
  3. **Mixed rooms:**
     - Claude, Codex and this session together.
     - Claude, Codex and Ollama together: all three answered the owner within 7 s. Agent-to-agent reviews worked (Ollama proposed and @-addressed the others; Claude objected; Codex filed a finding), and the reply-depth limit (`max_hops`) ended the chain at depth 3.
  4. **dispatchd** with Claude, Codex and Ollama, through a signed webhook (openssl/curl) and a one-off cron schedule. Each run made a closed room, seeded notes, got one typed finding per agent, revoked the invites and archived the room. The webhook run took 33 s; Claude's total cost was $0.067.

  Every live run found problems; all are fixed and recorded in the PRs above, lobby#10 and dispatch#4.
- **dispatchd** (room-o-matic/dispatch #1–#9) is done: templates, runs, cron, signed webhooks, operator API, the definitions API, workspaces on template workers, metrics, backup. lobbyd's new `service` key scope (lobby#10) lets operators get tokens for it, which the live test found was impossible before.
- **Claude Code as yourself** (client#15, #16; rooms#15; dispatch#6–#8):
  - `rom mcp` is an MCP server acting with **your own** lobbyd key (`claude mcp add rom -e ROM_API_KEY='${ROM_API_KEY}' …`), not a guest invite. Tools cover rooms, notes (compare-and-set), workers, dispatch runs and dispatch definitions. Room tools join on first use. All of it is live-verified with `claude -p` (Sonnet): it read the inbox and replied in a thread ($0.11), and it created a template, a webhook and a schedule from one plain-English request ($0.12). A delivery signed with the returned secret then ran an Ollama worker to `done`. The live run found that a new schedule showed `next_fire: null` until the next scheduler tick; fixed in dispatch#8.
  - `rom inbox --hook` is a `UserPromptSubmit` hook: messages that mention you, are addressed to you or reply to you are injected on each prompt, marked untrusted. It never blocks a prompt, even when misconfigured. The read position is in `~/.rom/inbox-<identity>.json`, shared with `rom inbox` and the `inbox_check` tool.
  - **dispatch definitions API:** templates, schedules and webhooks can be created over the operator API, stored in the database and validated with the config file as one set. File definitions are read-only there.
  - **Setup used in live tests** (in a scratch project dir): `claude mcp add -s project rom -e ROM_LOBBY_URL=… -e 'ROM_API_KEY=${ROM_API_KEY}' -e ROM_DISPATCH_URL=… -e ROM_STATE_DIR=… -- client/.venv/bin/rom mcp`, then a `.claude/settings.json` `UserPromptSubmit` hook running `rom inbox --hook`. Run with `claude -p --model sonnet --max-budget-usd 0.50 --mcp-config .mcp.json --strict-mcp-config --allowedTools "mcp__rom__*"`. The full recipe is in `client/README.md`.
  - **Not yet installed** in the owner's real Claude Code config: so far it's only been run in scratch projects.
- **Repos as knowledge bases** (client#17, dispatch#9, agents#25): `summon(workspace_path=)`, `rom summon --workspace`, MCP `worker_summon(workspace=)` and a dispatch worker's `workspace:` mount a directory on the agentd host, read-only under a `knowledge_read` profile. Summon skips an agentd that refuses the directory. The owner's candidates are `~/git/nomad_consul` and `~/git/openvpn`; don't modify them. `nomad_consul`'s working tree holds live cluster secrets (gitignored `secrets/`, `pki/`, `cluster.env`), so mount only a clean `git clone` of it, never the checkout.
  - **Clones live in `~/kb/openvpn` and `~/kb/nomad_consul`** (`/srv` is root-owned). Refresh them with `git -C ~/kb/<repo> pull`. Their committed files were scanned and hold no secrets; two UUID hits were a token *accessor* ID and an infrastructure note.
  - **Live-tested with Ollama** through `rom summon --workspace` and a dispatch template's `workspace:`. The original checkout and paths outside the roots were refused. The 7B model invented scripts until agents#26 added `search_files`, a first-turn orientation, a guard against invented file names and an empty-reply retry. It now cites real files, but its detail is unreliable. Use Claude for knowledge-base answers that matter (below).
  - **Live-tested with Claude (Sonnet, CLI 2.1.289):** both questions were answered correctly and fully grounded, for $0.068 and $0.080.
    - **openvpn:** Botrick is the preferred hub at 10.89.0.1/24 on `tun-botrick`; Boostie is the standby at 10.90.0.1/24, with hub-to-hub address 10.89.0.5. It cited line numbers.
    - **nomad_consul:** `sudo -E ./pki/30-revoke-friend.sh <name>` (or `make revoke`), then `nomad acl token delete …`, then verify from outside. It added caveats on the CRL timer and the `probe` identity.
    - Every claim was checked against the files. Claude used Grep/Glob/Read only, confined by `--restricted`. Prefer Claude over the 7B Ollama model for knowledge-base answers.
  - **Recommended for real use:** `backend: sandbox` for these sessions. The process backend only sets the working directory; it doesn't confine a worker's own tools.
- **Agents without the stack** (agents#28): `agentd ask <agent> <question>` and `agentd mcp` (tools `agents_list`, `agent_ask`) run a configured agent from a local agents file (`~/.config/agentd/agents.yaml`, `$AGENTD_AGENTS`; example `agents/agents.local.example.yaml`). No lobbyd, roomsd or agentd service, nothing in the background; each ask is a fresh worker, ended (whole process group) after its first answer. The owner asked for this: their own Claude asks agents, and nothing runs when unused.
  - **Live-tested** with no services running:
    - **Ollama knowledge-base agent:** process backend (60 s) and sandbox (15 s).
    - **Claude, via the CLI:** the `nomad` agent gave the full revocation procedure from 8 cited files ($0.076, 17 s).
    - **Claude, via the installed MCP tools in a real `claude -p` session:** the session listed the agents, picked `openvpn` and relayed a correct hub answer (outer session $0.095).
    - **Confinement:** a probe confirmed `--restricted` refuses Read/Glob outside the workspace.
  - **Sandbox rule:** a model-backed worker needs `network: true` to reach its model; `network: false` unshares the network, so it can't even reach Ollama on localhost. With network on, `claude_tools: [Read, Grep, Glob]` keeps Claude read-only. Sandboxed Claude also needs credentials it can see (private HOME), e.g. `ANTHROPIC_API_KEY` in `env_allowlist`.
  - **Installed for the owner** (2026-10-05):
    - **Agents file:** `~/.config/agentd/agents.yaml` (mode 600) defines `openvpn` and `nomad` (Claude Sonnet, $0.25 cap per ask) and `quick` (Ollama), all on `knowledge_read` over the `~/kb` clones, with `backend: process`.
    - **MCP server:** user-scope `agents` in `~/.claude.json`, running `agents/.venv/bin/agentd mcp`. It runs from this checkout, so `git pull` plus `uv sync` in `agents/` updates it.
  - **Counting leftover workers:** don't `ps | grep 'python.*-m agentd.workers'` from a command line that itself contains both strings; it matches its own shell.
- **Next: the owner's real agents (Odin, Boostie, Missy on OpenClaw) across machines.** Postponed by the owner in favour of the Claude Code work above. Two gaps:
  - **OpenClaw integration:** not built. `PeerAgent` and `design/peer-protocol.md` exist; ask how the bots are built before starting.
  - **Deployment packaging:** none yet. Needs systemd units, or resuming the paused Docker work, plus TLS and stable canonical URLs.
- **Resolved follow-ups from live testing** (all live-verified in one sequential dispatch run):
  - **Workers raced** → `run.order: sequential` (dispatch#5): each worker is summoned after the previous one ends, so later workers build on earlier posts.
  - **The 7B model omitted confidence** → `rooms_send` parameter descriptions (agents#24): measured 3/12 → 12/12. Model choice guidance is in `agentd.example.yaml`; small local models still make factual slips.
  - **`rom say` couldn't thread** → `--reply-to`, `--to` and `--reply-requested` (client#14).
  - **lobby `test_operations` flake** → it was a real CLI bug (lobby#11): about 1 in 64 kids started with `-`, so `signing-key retire <kid>` failed. New kids start with `k`, and older ones work with `retire -- <kid>`.
- **Running live tests.** Stand the stack up in a scratch dir, as in the docs README quickstart. Ports: lobbyd 8767, roomsd 8766, agentd 8765, dispatchd 8768.
  - **agentd worker types:**
    - `claude` / `claude-chat`: `--model sonnet --max-budget-usd …`
    - `codex`: `--model gpt-6-sol --max-total-tokens …`
    - `ollama`: `--model qwen2.5:7b-instruct-q4_K_M`
  - **Grants:** give `you@local` (or `dispatch@local`) a `callers` grant at agentd.
  - **Knowledge bases:** `workspace_roots: [~/kb]`, a profile `knowledge_read: {max_runtime_minutes: 10, workspace_mount: read, network: false, filesystem: read}`, and the same `workspace_roots` in the caller's grant.
  - **Proving a repo is untouched:** snapshot HEAD, `GIT_OPTIONAL_LOCKS=0 git status --porcelain --ignored`, and every file's mtime and size, before and after. Without `GIT_OPTIONAL_LOCKS=0`, `git status` refreshes `.git/index` and the check itself changes the repo.
  - **For dispatch:** a template using oneshot worker types, a `HOOK_*` secret env var, and `lobbyd key create dispatchd --scope service --endpoint http://127.0.0.1:8768` plus `DISPATCHD_OPERATORS` for the operator API.
  - **Wake-ups:** workers are woken by `@<summon --name>`, or `@<name>-<run suffix>` for dispatch.
  - **Cost:** Claude runs cost money and Codex uses the ChatGPT plan, so ask before each run and cap budgets. Ollama is free.
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
- **`service` keys** (lobby#10) anchor an approved endpoint for any other service that authenticates callers with lobbyd tokens, such as dispatchd's operator API. They need a `can_host` tenant, and `apikeys.HOSTED` lists every endpoint-owning scope. The key itself is inert: it can't exchange tokens or register, and its endpoint isn't in the directory. A new service that accepts lobbyd tokens needs one, or lobbyd won't mint tokens for it.
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
- `GET /v1/me/updates?cursor=` returns new messages across every room the caller has joined (for an agent: every room it holds `read` in, joined or not, rooms#15; an invite: its one room). It works because `messages.id` is a single autoincrement across the whole server; keep it that way.
- Listing: `listed`/`tags` are set on create or by the creator through `PATCH /v1/rooms/{id}`. `lobby_client.sync_loop` (started in the lifespan only when `ROOMSD_LOBBYD_API_KEY` is set) heartbeats the server into lobbyd every ttl/3. It also pushes any room whose `listing_version > listing_synced_version`. Routes bump `listing_version` and wake the loop through `app.state.lobby_wake`. Never write to lobbyd from a route.
- `db.py` (all four services, docs#24): the schema version is SQLite's `user_version`.
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
- `workers/codex.py` is the **Codex CLI adapter**. It runs one `codex exec --json` per turn; later turns `resume` the thread, and prompts go in on stdin. It reuses the Claude adapter's framing, system prompt, `RoomWatcher`, wake gate and closing summary.
  - **Permissions:** the profile picks Codex's sandbox (`read-only` or `workspace-write`; network only if allowed). It runs with `approval_policy="never"`, `--ignore-user-config` and `shell_environment_policy.inherit="core"`. Use `--external-sandbox` only on `backend: sandbox`.
  - **Room tools:** the server gets its token by name (`env_vars`), never in argv. Only that server is pre-approved (`default_tools_approval_mode="approve"`); without that, Codex refuses every MCP call.
  - **Budget:** `--max-total-tokens` / `max_total_tokens` counts uncached input plus output, because Codex resends the thread every turn.
  - **Turns must end.** The shared instructions forbid waiting or polling inside a turn. Without that rule, Codex told to "wait" kept its turn open, and every later message queued behind it. `--max-turn-seconds` (default 600) stops a runaway turn; an interactive session keeps listening.
  - **Test fake:** `tests/fake_codex.py` emits JSONL captured from a real run. When the Codex CLI's event format changes, update `Translator` and the fake together.
- `local.py` is the **standalone runner** behind `agentd ask` / `agentd mcp`: `AgentsFile` (agents file model, reusing `WorkerType`/`Profile`), `ask()` and `AskTools`. It builds the env with `supervisor.worker_environment()` (shared with the gateway; change both callers together) and uses the same backends. It returns the first `final`, or an interactive worker's `needs_input`, and ends the group without a polite stop when it has an answer, so no closing-summary turn is spent.
- `workers/turns.py` (`TurnAdapter`) is the session loop for turn-based adapters: Codex and Ollama subclass it and supply `run_turn`. It handles oneshot or interactive sessions, framing, wakes, budget, the turn time limit, stop and the closing summary. Fix session behaviour there, not per adapter.
- `workers/ollama.py` is the **Ollama adapter** for local models. Ollama only serves models, so the adapter is the agent loop: `/api/chat` with tools, which it runs itself.
  - **Tools:** room tools in-process, with the MCP server's schemas; `read_file`/`list_files`/`search_files` for a read or read_write workspace; `write_artifact` for read_write. No shell, no web. `search_files` matches lines containing every query word and skips `.git`, binaries and symlinks out of the workspace.
  - **Small-model guards:**
    - Exact repeat posts are refused, arguments are coerced to the schema, and tool results are plain sentences.
    - With a workspace, the first turn gets an orientation: the top-level listing, the README head, and the other guide docs.
    - **Invented-file guard (`Toolbox.unknown_files`):** a post or final answer naming a repo path that neither exists nor appears in the workspace's text is sent back once. A multi-segment path is checked only if its first directory exists at the top level.
    - An empty reply gets one retry before the session fails.
  - **Logging:** building the MCP server sets root logging to INFO, so the adapter resets it to WARNING.
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
uv run lobbyd key create <name> [--scope agent|agentd|roomsd|service]   # long-lived API key
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
  - `summon` = pick an agentd instance → invite → spawn. It **revokes the invite if the spawn fails**. `workspace_path` is a directory on the agentd host. When summon picks the instance, one that refuses the workspace (`refuses_workspace`: a 403/422 about the workspace) is skipped like a full one.
  - `summon` always has an `operation_id` (docs#13). After an ambiguous failure it reconciles through agentd's `by-operation` lookup before revoking or retrying. Invites outlive the profile's maximum runtime plus 5 minutes (`invite_ttl_for`, docs#17).
  - `finalize_session` revokes a worker's invite by id when agentd hands the close-out back (`owner_required`).
  - `cli.fmt_message` prints typed fields as `[re:#N to:… conf=… severity=… reply-requested]`; keep it in step with `PAYLOAD_FIELDS`.
  - `inbox.py` / `mcp.py`: `rom inbox [--hook]` and `rom mcp` for a session acting as you (see Current state). New MCP tools go in `TOOLS` and the README table, and must raise `ToolError` (via `readable_errors`) so the session sees the reason.
  - `watch` polls `/v1/me/updates` on each server. Its "from now" cursor is resolved when `watch()` is called, not on first iteration.
- Service API changes need matching updates here, in the FakeWorld fakes, and in `scripts/e2e.py`.

## dispatchd commands and layout (run from `dispatch/`)

```bash
uv sync && uv run pytest -q
uv run python scripts/e2e.py              # real lobbyd + roomsd + agentd from the siblings
export DISPATCHD_CONFIG=dispatch.yaml DISPATCHD_DATA_DIR=.data LOBBYD_URL=… DISPATCHD_LOBBYD_API_KEY=…
uv run dispatchd check-config | serve | schedules | run <schedule> | runs | backup …
```

- **What it is:** scheduled and webhook-triggered rooms. dispatchd is an ordinary agent identity using the `roomomatic` client, never part of lobbyd, which calls nobody. Its power is bounded by agentd's caller grant for it and by room rights. The design is in `design/dispatch.md`.
- **Modules:**
  - `config.py`: templates (the restriction), schedules and webhooks. Strict: unknown keys are rejected.
  - `runner.py`: the run lifecycle. Each step is recorded, so a resumed run never duplicates; summons use `operation_id=<run>.<worker>` and offers use `offer_id=<run>.<agent>`.
  - `scheduler.py`: no fire on first sight; `schedule_state.spec` triggers recompute after an edit; a missed fire beyond `catch_up` is skipped.
  - `hooks.py`: HMAC over `"<ts>.<body>"`, plus delivery dedupe.
  - `app.py`: the loop, run threads, webhook and operator routes. `LiveDefinitions` merges the file with the API's definitions.
  - `store.py`: API-created definitions (`definitions` table, schema v3). `merge` validates file + API as one set; a name in both is an error. API webhooks get a generated `whsec_` secret, returned only on create or rotate. A retired secret is journaled by SHA-256 fingerprint (`revocations.jsonl`) before the change commits, and `recovery.post_restore` clears matching secrets.
- **`rules` aren't enforced:** they're text. Enforcement comes from agentd profiles and grants, room admission and rights, and loop guards.
- **Worker handles** are `<name>-<run suffix>`, because a guest identity has at most one live invite.
- **A template worker can name `workspace:`** (an absolute path on the agentd host), which is passed to `summon(workspace_path=)`.
- **dispatch depends on `roomomatic` as a git dependency.** After a client change dispatch needs, run `uv lock --upgrade-package roomomatic` in `dispatch/` and commit the lock.

## How the services relate

Dependencies run one way: roomsd, agentd and agents call lobbyd; agentd and its workers call roomsd; dispatchd calls all three as an ordinary agent identity; **lobbyd calls nobody, and roomsd never calls agentd**. Agents normally go through the `roomomatic` client, which calls all three.

- **roomsd**: peer collaboration. Independent agents (Boostie, Missy, Claude, Odin, …) that already exist share durable rooms. It **never** spawns or schedules agents, makes model calls, runs tools for agents, or decides who is right. A room holds a chat log of typed messages, notes (blackboard state), tasks with lease-based claims, an artifact index, and a decision log.
- **agentd**: worker orchestration. An always-on gateway that lazily spawns ephemeral helper workers (process at first, Docker later). Callers talk to a *session*, not a shell process. It owns lifecycle: spawn, message, SSE events, status, stop, idle and hard-timeout cleanup.
- **lobbyd**: identity issuer plus directory of roomsd servers, agentd instances and listed rooms. It never sees messages or sessions.
- **dispatchd**: opens rooms on a schedule or a signed webhook: it creates the room, brings in workers and peers under a template's restrictions, sets the goal, waits, and archives.

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
