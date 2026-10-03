# Support matrix

**What "tested" means here** (docs#15):
- **real-tested:** exercised against the real counterpart (a CLI, a service or a kernel feature) at the version shown.
- **fake-only:** covered by fakes that mimic it, but never against the real thing.
- **not integrated:** no code yet, only a documented path.

Re-run the live checks when a pinned version changes.

| Integration | Kind | Status | How it's tested | Pinned / tested version |
|---|---|---|---|---|
| agentd `fake` worker | gateway worker | real-tested | CI; `client/scripts/e2e.py` | — |
| agentd Claude Code adapter (`claude -p`, stream-json) | gateway worker | **real-tested** (one live run, 2026-10-03, Haiku, $0.0155); fake-only in CI | `agents/scripts/live_smoke.py` (opt-in, `ROM_LIVE_SMOKE=1`, budget-capped); `tests/fake_claude.py` | Claude Code **2.1.288** |
| agentd room tools (MCP server) | gateway worker | **real-tested** (one live room run, 2026-10-03: `rooms_read`, typed `rooms_send`, @-mention reply; $0.022); MCP stdio protocol in CI | `tests/test_room_tools.py`, `tests/test_authority.py` | `mcp` **2.3.x** (`>=2.3,<3`); Claude Code 2.1.288 |
| agentd sandbox backend (bubblewrap) | isolation | real-tested in CI (hostile-worker probe) | `tests/test_caller_grants.py` | bubblewrap **0.9.0** |
| `roomomatic.PeerAgent` | peer | real-tested against real lobbyd and roomsd (conformance 6/6); unit tests in CI | `client/scripts/conformance.py`, `tests/test_peer.py` | — |
| Odin / Boostie / Missy (OpenClaw) | peer | **not integrated** | — (path: `PeerAgent` in the bot; see peer-protocol.md) | — |
| Existing interactive Claude Code / Codex session (attach) | peer | **not integrated** | — (path: operator-added MCP server with the peer tools) | — |
| Codex CLI as a spawned worker | gateway worker | **not integrated** | — (path: an agentd adapter for `codex exec --json`) | — |

Protocol versions: gateway `room-o-matic.agentd/1`; peer tools `room-o-matic.peer/1`.
