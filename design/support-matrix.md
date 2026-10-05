# Support matrix

**What "tested" means here** (docs#15):
- **real-tested:** exercised against the real counterpart (a CLI, a service or a kernel feature) at the version shown.
- **fake-only:** covered by fakes that mimic it, but never against the real thing.
- **not integrated:** no code yet, only a documented path.

Re-run the live checks when a pinned version changes.

| Integration | Kind | Status | How it's tested | Pinned / tested version |
|---|---|---|---|---|
| agentd `fake` worker | gateway worker | real-tested | CI; `client/scripts/e2e.py` | — |
| agentd Claude Code adapter (`claude -p`, stream-json) | gateway worker | **real-tested** (live runs 2026-10-03/04: Haiku and Sonnet, rooms, mixed rooms, dispatch; 2026-10-05: read-only knowledge-base workspaces); fake-only in CI | `agents/scripts/live_smoke.py` (opt-in, `ROM_LIVE_SMOKE=1`, budget-capped); `tests/fake_claude.py` | Claude Code **2.1.288**, **2.1.289** |
| agentd room tools (MCP server) | gateway worker | **real-tested** (one live room run, 2026-10-03: `rooms_read`, typed `rooms_send`, @-mention reply; $0.022); MCP stdio protocol in CI | `tests/test_room_tools.py`, `tests/test_authority.py` | `mcp` **2.3.x** (`>=2.3,<3`); Claude Code 2.1.288 |
| agentd sandbox backend (bubblewrap) | isolation | real-tested in CI (hostile-worker probe) | `tests/test_caller_grants.py` | bubblewrap **0.9.0** |
| `roomomatic.PeerAgent` | peer | real-tested against real lobbyd and roomsd (conformance 6/6); unit tests in CI | `client/scripts/conformance.py`, `tests/test_peer.py` | — |
| dispatchd schedules, webhooks and definitions API | automation | **real-tested**: live runs with Claude, Codex and Ollama workers (signed webhook, cron schedule, sequential order, workspace mounts); `dispatch/scripts/e2e.py` against real services with the fake worker; unit tests in CI | `tests/test_*.py`, `scripts/e2e.py` | — |
| Odin / Boostie / Missy (OpenClaw) | peer | **not integrated** | — (path: `PeerAgent` in the bot; see peer-protocol.md) | — |
| Your own interactive Claude Code session (`rom mcp` + `rom inbox --hook`) | client | **real-tested** (`claude -p`, Sonnet, 2026-10-04: inbox reply in a thread; creating dispatch templates, webhooks and schedules); unit tests in CI | `client/tests/test_rom_mcp.py`, `client/scripts/e2e.py` | Claude Code 2.1.288 |
| Interactive session as a *peer* (offers through lobbyd) | peer | **not integrated** | — (`rom mcp` acts in rooms as you, but has no offer tools) | — |
| Backup / restore / schema upgrade (lobbyd, roomsd, agentd, dispatchd) | operations | real-tested in CI on disposable copies (docs#24) | `tests/test_operations.py` in each repo; `design/operations.md` drills | SQLite **3.45.1** (online backup API) |
| agentd Codex CLI adapter (`codex exec --json`) | gateway worker | **real-tested** (one live room run, 2026-10-04, `gpt-6-sol` via ChatGPT login); fake-only in CI | `tests/test_codex_adapter.py`, `tests/fake_codex.py` (events captured from the real CLI) | Codex CLI **0.158.0** |
| agentd Ollama adapter (local models via `/api/chat` with tools) | gateway worker | **real-tested** (live room runs 2026-10-04, `qwen2.5:7b-instruct-q4_K_M` on a 6 GB GPU; knowledge-base runs: grounded in real files, detail unreliable at 7B); fake-only in CI | `tests/test_ollama_adapter.py`, `tests/fake_ollama.py` (response shape captured from real Ollama) | Ollama **0.10.1** |

Protocol versions: gateway `room-o-matic.agentd/1`; peer tools `room-o-matic.peer/1`.
