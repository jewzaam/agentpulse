# AgentPulse

Backend service that collects Claude Code agent state and exposes it via REST + WebSocket.

## Commands

Targets follow **verb‑[qualifier‑]noun**. The root Makefile holds standard
targets (`help`, `check`, `clean`, `format`, `install*`, `uninstall*`, `run`)
plus project-mutated composites (`install`, `uninstall`, `run`). Subsystem
targets live in noun-named fragments: `make/hooks.mk`, `make/service.mk`,
`make/tray.mk`. The standard `test-*` collection lives in `make/test.mk`.
Run `make help` for the full sorted list.

```bash
make check                       # Full quality gate (default)
make format                      # Rewrite sources with black
make test-unit                   # Run pytest only

make install                     # Install everything: pipx + hooks + service autostart
make uninstall                   # Inverse of install (also tears down tray autostart)
make install-dev                 # Editable venv install + dev + tray extras
make install-pipx                # Install globally via pipx (component of `install`)

make run                         # Run service + tray in foreground (Ctrl-C exits both)
make run-service                 # Service only
make run-tray                    # Tray only

make install-hooks               # Component: Claude Code hooks
make install-autostart-service   # Component: service auto-start on login
make install-autostart-tray      # Component: tray auto-start (opt-in; not part of `make install`)
```

## Configuration

All configuration via `~/.claude/agentpulse/config.json` (`--config` is required on CLI).
See `config.example.json` for the schema. No environment variable overrides.

Key settings: `host`, `port`, `db_path`, `log_file`, `log_level`, `discovery_interval_seconds`,
`fetch_limits` (bool — enables OAuth API usage limit fetching).

## Hook Setup

`make install-hooks` copies the relay script and **auto-merges** hook entries into
`~/.claude/settings.json` (deep merge, idempotent — replaces existing agentpulse entries
in-place, preserves other hooks).

The relay script (`scripts/hook_relay.py`) reads JSON from stdin, injects `source_system`
(hostname) and `pid` (parent process = Claude Code), and POSTs to agentpulse.
Reads host/port from the same config file. The PID enables liveness detection and
entrypoint inference (VS Code, Cursor, terminal) via process tree inspection.

## Statusline Integration

Claude Code's statusline sends per-session cost/token/context data via stdin JSON.
`my-claude-stuff/statusline.py` relays this to `POST /statusline/claude` before its
own processing. AgentPulse stores each update in `claude_statusline` (append-only history
with extracted numerics) and enriches `claude_sessions` with the latest values.

## Architecture

```
src/agentpulse/
├── app.py                        # FastAPI factory + lifespan (discovery loop)
├── config.py                     # Settings from config file (--config required)
├── db.py                         # SQLite singleton (aiosqlite, WAL mode)
├── models.py                     # Normalized Pydantic response models
├── websocket.py                  # ConnectionManager + /ws endpoint
├── platforms/claude/
│   ├── hooks.py                  # POST /hooks/claude, /statusline/claude, /costs/claude
│   ├── discovery.py              # PID liveness checks + entrypoint detection
│   ├── limits.py                 # OAuth API usage limit fetching + DB cache
│   ├── schema.py                 # claude_* table DDL + all CRUD queries
│   └── models.py                 # Claude-specific Pydantic models + derive_state()
└── api/
    ├── sessions.py               # Normalized endpoints: /api/v1/sessions, /summary, /limits
    └── claude.py                 # Raw endpoints: /api/v1/claude/sessions, /events
```

## Key Design Decisions

- **Raw events stored, state derived at query time** — DB has `last_event` and `last_tool`,
  not a `state` column. `derive_state()` in `platforms/claude/models.py` computes state
  strings at the API layer. Do not store computed states.
- **Platform-scoped tables** — All tables prefixed `claude_` (e.g., `claude_sessions`,
  `claude_events`, `claude_statusline`, `claude_limits`). Future platforms get their own
  tables. Normalization happens in `api/sessions.py`, not in storage.
- **REST responses and WebSocket broadcasts must match** — every data field on a
  session or event in the REST API must also appear in the corresponding WebSocket
  broadcast. Consumers should be able to build the same view from either source.
- **Config file is the only configuration source** — no env vars. Both the service
  (`--config`) and the hook relay (`--config`) read the same file.
- **Limits fetching is optional** — `fetch_limits: true` enables OAuth API calls.
  When false, completely inert (no token read, no API call). Inert on Vertex.

## WebSocket API

**Endpoint:** `ws://host:port/ws` — receive-only JSON text frames.

**Message types:** `session_discovered`, `session_ended`, `session_cleared`, `hook_event`,
`agent_started`, `agent_stopped`, `statusline_update`, `limits_updated`. Every message
has `type`, `platform`, and `timestamp`. Most have `session_id`.

**Client pattern:** Bootstrap via `GET /api/v1/sessions` on connect, then apply WebSocket
updates incrementally by switching on `message.type`.

**Full client guide:** [docs/design/clients.md](docs/design/clients.md) — all message types,
shapes, REST endpoints, derived state mapping, and client implementation notes.

**Ping/pong:** AgentPulse does NOT send WebSocket pings. Clients that expect server-initiated
pings will timeout and disconnect. Configure the client to either disable ping expectations
or send its own keep-alive pings.

## Testing

- `tests/conftest.py` has autouse safety guards: blocks subprocess, filesystem writes
  outside tmp_path, real HTTP (including httpx.Client/AsyncClient), and creates an
  isolated config file pointing DB at tmp_path
- Each test file creates its own async `db` fixture via `init_db(db_path=tmp_path / "test.db")`
- Use `asyncio_mode = "auto"` — async test functions run automatically
- API tests use `FastAPI TestClient` with routers wired directly (no lifespan)
- Sync test methods that need async DB calls use `run_async()` from conftest

## Gotchas

- `db.py` uses a module-level singleton (`_db`). Tests must call `close_db()` in teardown
  to avoid leaking connections across test files.
- `validate_pid()` and `detect_entrypoint()` are sync and wrapped in
  `asyncio.to_thread()`. Tests mock psutil to avoid hitting real PIDs.
- The `src/` layout means Makefile targets use `SRC_DIR = src/$(PACKAGE_NAME)` for format,
  lint, typecheck — not the bare package name.
- Windows `ProactorEventLoop` raises `ConnectionResetError` when hook relay clients
  disconnect early. Suppressed by a logging filter in `app.py` on the `asyncio` logger.
- `aiosqlite` debug logging is suppressed (set to WARNING) to keep logs readable.
- Schema changes require dropping the DB if tables already exist — no migration system yet.
  `CREATE TABLE IF NOT EXISTS` won't add new columns to existing tables.
