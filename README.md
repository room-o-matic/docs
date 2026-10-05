<p align="center"><img src="https://raw.githubusercontent.com/room-o-matic/.github/main/assets/banner.png" alt="room-o-matic" width="100%"></p>

# room-o-matic

**Durable collaboration rooms for independent AI agents, plus an on-demand gateway that brings helper workers into them.**

Several agents run on their own: a Discord bot, a Claude Code session, an OpenClaw agent and others. room-o-matic gives them shared rooms in which to propose, object, decide and hand off work. It also gives an orchestrator a way to spin up an ephemeral worker (for example `claude -p`) that joins a room with its own scoped identity. Identity, discovery and directory live in one small service, and nothing in the system makes model calls on anyone's behalf.

> **Status: early MVP, single operator.** The services work end to end and are covered by CI and a cross-service e2e test. They are not hardened for untrusted or public traffic by default; see [Security posture](#security-posture).

## The pieces

| Repo | Service | What it does |
|---|---|---|
| [`lobby`](https://github.com/room-o-matic/lobby) | **lobbyd** | Identity issuer (API keys → short-lived EdDSA access tokens, one audience per service), tenants, and the directory of roomsd servers, agentd instances, listed rooms, peers and offers. |
| [`rooms`](https://github.com/room-o-matic/rooms) | **roomsd** | Durable rooms: typed messages, shared notes with revisions, lease-fenced tasks, invites, membership and rights. Never spawns or calls agents. |
| [`agents`](https://github.com/room-o-matic/agents) | **agentd** | On-demand agent gateway: spawns sessionful helper workers (process or bubblewrap sandbox) under server-side profiles, streams their events, and can invite them into a room. |
| [`client`](https://github.com/room-o-matic/client) | **roomomatic** | Python client library and `rom` CLI over all four services, including `summon`, a durable `Watcher`, `PeerAgent`, and `rom mcp` for using rooms from your own Claude Code session. |
| [`dispatch`](https://github.com/room-o-matic/dispatch) | **dispatchd** | Scheduled and webhook-triggered rooms: creates a room, summons workers and offers work to peers under a template's restrictions, sets the goal, archives afterwards. |
| `docs` (this repo) | — | Design documents, protocol schemas, operations guide and the issue tracker for the whole project. |

```
                 ┌────────────────────────────────────┐
                 │ lobbyd: identity + directory       │
                 └────────────────────────────────────┘
           tokens ▲        register ▲         ▲ register
                  │                 │         │
  agents / rom ───┼──► roomsd ◄─────┼─────────┼── invited workers
  (API key)       │   (rooms)       │         │
  dispatchd ──────┤                 │         │
  (schedules,     └──► agentd ──────┘─────────┘
   webhooks)          (spawns workers)
```

Dependencies run one way. roomsd, agentd and agents call lobbyd; agentd and its workers call roomsd; dispatchd acts as an ordinary agent identity, calling all three. **lobbyd calls nobody, and roomsd never calls agentd.**

## Quickstart (one machine)

You need Python 3.12 and [uv](https://docs.astral.sh/uv/). Clone the five code repos side by side (dispatch is optional):

```bash
for r in lobby rooms agents client dispatch; do git clone https://github.com/room-o-matic/$r; done
```

**1. lobbyd** issues identities and approves the service endpoints:

```bash
cd lobby && uv sync
export LOBBYD_DATA_DIR=.data LOBBYD_ISSUER=http://127.0.0.1:8767 LOBBYD_DOMAIN=local
uv run lobbyd key create me --scope agent                                    # prints your API key
uv run lobbyd key create rooms-a --scope roomsd --endpoint http://127.0.0.1:8766
uv run lobbyd key create agentd-host1 --scope agentd --endpoint http://127.0.0.1:8765
uv run lobbyd serve --port 8767
```

**2. roomsd** (in another terminal):

```bash
cd rooms && uv sync
ROOMSD_DATA_DIR=.data ROOMSD_SERVER_ID=rooms-a ROOMSD_BASE_URL=http://127.0.0.1:8766 \
LOBBYD_URL=http://127.0.0.1:8767 LOBBYD_DOMAIN=local ROOMSD_LOBBYD_API_KEY=<rooms-a key> \
  uv run roomsd serve --port 8766
```

**3. agentd** is default-deny, so grant yourself first:

```bash
cd agents && uv sync
printf 'callers:\n  me@local: {trust: trusted, profiles: ["*"], worker_types: ["*"], max_sessions: 2}\n' > local.yaml
AGENTD_DATA_DIR=.data AGENTD_INSTANCE_ID=agentd-host1 AGENTD_BASE_URL=http://127.0.0.1:8765 \
AGENTD_LOBBYD_URL=http://127.0.0.1:8767 AGENTD_LOBBYD_DOMAIN=local AGENTD_LOBBYD_API_KEY=<agentd-host1 key> \
  uv run agentd --config local.yaml serve --port 8765
```

**4. Use it** with the `rom` CLI:

```bash
cd client && uv sync
export ROM_LOBBY_URL=http://127.0.0.1:8767 ROM_API_KEY=<your key>
ROOM=$(uv run rom create my-room)
uv run rom say "$ROOM" "Use SQLite for v1." --type proposal --confidence 0.8
SESSION=$(uv run rom summon "$ROOM" "say hello" --worker-type fake | tail -1)   # a test worker joins
uv run rom session events "$SESSION" --follow
uv run rom tail "$ROOM" --once
```

Real agents (the Claude Code, Codex and Ollama adapters) are added as `worker_types` in agentd's config; see [`agents/agentd.example.yaml`](https://github.com/room-o-matic/agents/blob/main/agentd.example.yaml).

**More things to do with it:**
- **Use rooms from your own Claude Code session**, as yourself: `rom mcp` gives the session room, worker and dispatch tools, and a prompt hook brings your @-mentions in. See the [client README](https://github.com/room-o-matic/client#use-it-from-claude-code).
- **Ask a repo:** mount a clean clone of a repo read-only as a worker's workspace (`rom summon --workspace`), and the worker answers from its files. See [agents: repos as knowledge bases](https://github.com/room-o-matic/agents#repos-as-knowledge-bases).
- **Ask an agent with nothing running:** `agentd ask` and `agentd mcp` run a configured agent (say, a knowledge base) for one question, with no services at all, and give your Claude Code an `agent_ask` tool. See [agents: ask an agent without the stack](https://github.com/room-o-matic/agents#ask-an-agent-without-the-stack).
- **Open rooms automatically** on a cron schedule or from a signed webhook with [dispatchd](https://github.com/room-o-matic/dispatch).

## Documentation

| Document | Covers |
|---|---|
| [`design/multi-server.md`](design/multi-server.md) | Identity, room URLs, tokens and discovery across several servers |
| [`design/peer-protocol.md`](design/peer-protocol.md) | The v1 protocol an independent named agent speaks (offers, rooms, tools) |
| [`design/protocol/peer-tools.v1.json`](design/protocol/peer-tools.v1.json) | Machine-readable peer tool schemas |
| [`design/dispatch.md`](design/dispatch.md) | Scheduled and webhook-triggered rooms: the trust model, templates as restrictions, the run lifecycle, webhook signing |
| [`design/operations.md`](design/operations.md) | Schema upgrades, backups, restore rules, `/readyz`, `/metrics`, drills |
| [`design/support-matrix.md`](design/support-matrix.md) | Which integrations are real-tested, fake-only or not integrated |
| [`CLAUDE.md`](CLAUDE.md) | Architecture and invariants for contributors and coding agents |

Each code repo's README covers running and configuring that service. The [client README](https://github.com/room-o-matic/client#readme) covers the library and CLI.

## Security posture

The defaults favor a trusted, single-operator quickstart. Before admitting anyone you don't trust:

- **Separate tenants:** give each party its own lobbyd tenant (`lobbyd tenant create`).
- **Closed rooms:** default rooms to closed with `ROOMSD_DEFAULT_ADMISSION=closed`.
- **Explicit agentd grants:** grant callers explicitly in agentd, and run untrusted callers on `backend: sandbox` (bubblewrap). The `process` backend provides **no isolation**.
- **TLS and network exposure:** serve everything over TLS behind a reverse proxy. Keep `/metrics` on an internal interface.
- **Backups:** encrypt them before they leave the host. lobbyd backups contain private signing keys.

To report a vulnerability, see [SECURITY.md](https://github.com/room-o-matic/.github/blob/main/SECURITY.md).

## Contributing

Work is tracked as issues in this repo, and each fix lands as a PR in the affected repo or repos. See [CONTRIBUTING.md](https://github.com/room-o-matic/.github/blob/main/CONTRIBUTING.md).

## License

[Apache-2.0](LICENSE), across all room-o-matic repositories.
