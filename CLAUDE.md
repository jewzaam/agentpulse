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

v1 and v2 coexist during the rollout. Hooks and statuslines dual-write
to both layers. v1 endpoints/tables retire in slice E. See
`docs/old/changes-with-v2.md` for the slicing plan and `plan-v2-refactor.md`
for current slice status.

```
src/agentpulse/
├── app.py                        # FastAPI factory + lifespan (discovery + pid watcher loops)
├── config.py                     # Settings from config file (--config required)
├── db.py                         # SQLite singleton (aiosqlite, WAL mode)
├── models.py                     # v1 normalized Pydantic response models
├── websocket.py                  # v1 ConnectionManager + /ws endpoint
├── events.py                     # v1 broadcast functions
├── platforms/claude/
│   ├── hooks.py                  # POST /hooks/claude, /statusline/claude, /costs/claude (dual-writes v1+v2)
│   ├── discovery.py              # v1 session-level PID liveness + entrypoint detection
│   ├── pid_watcher.py            # v2 PID death watcher → claude_log_pid_deaths
│   ├── limits.py                 # OAuth API usage limit fetching + DB cache
│   ├── schema.py                 # claude_* DDL + queries (v1 + v2 log_* tables, inserters, replay)
│   └── models.py                 # Claude payload models + v1 derive_state()
└── api/
    ├── sessions.py               # v1: /api/v1/sessions, /summary, /limits
    ├── claude.py                 # v1: /api/v1/claude/sessions, /events
    └── v2/
        ├── router.py             # /api/v2/processes, /sessions, /log/{hooks,statuslines,pid-deaths}
        ├── models.py             # ProcessResponse, SessionResponse, EpochResponse, ...
        ├── ids.py                # process_id, epoch_id (sha256 prefix-16)
        ├── state.py              # v2 derive_state (Stop → ready) + compute_effective_state
        ├── websocket.py          # /ws/v2 endpoint (separate ConnectionManager from v1)
        ├── events.py             # broadcast_hook_logged, statusline_logged, pid_death_logged
        └── queries/              # projection layer over claude_log_* tables
            ├── log.py            # raw filtered reads
            ├── enrich.py         # IdEnricher (per-request id cache)
            ├── sessions.py       # session/epoch derivation
            ├── processes.py      # process derivation (consults pid_deaths for ended_at/pid_alive)
            └── _helpers.py       # shared latest-hook + agent helpers
```

## Key Design Decisions

- **Raw events stored, state derived at query time** — DB has `last_event` and `last_tool`,
  not a `state` column. `derive_state()` in `platforms/claude/models.py` computes state
  strings at the API layer. Do not store computed states.
- **Platform-scoped tables** — All tables prefixed `claude_`. v1 tables:
  `claude_sessions`, `claude_agents`, `claude_events`, `claude_statusline`,
  `claude_costs`, `claude_limits`. v2 append-only log tables:
  `claude_log_hooks`, `claude_log_statuslines`, `claude_log_pid_deaths`,
  `claude_log_api_limits`. Future platforms get their own tables.
  Normalization happens at the API layer, not in storage.
- **v2 inserters return the row id** — `insert_log_hook`, `insert_log_statusline`
  return `int | None` (None when payload lacks `pid`). `insert_log_pid_death`
  returns `int` (always inserts). `insert_log_api_limits` returns
  `(int, dict)` — the row id plus the normalized fields the broadcast
  needs. Callers use the returned id to broadcast `*_logged` frames on
  `/ws/v2` after the dual-write commits.
- **REST responses and WebSocket broadcasts must match** — every data field on a
  session or event in the REST API must also appear in the corresponding WebSocket
  broadcast. Consumers should be able to build the same view from either source.
- **Config file is the only configuration source** — no env vars. Both the service
  (`--config`) and the hook relay (`--config`) read the same file.
- **Limits fetching is optional** — `fetch_limits: true` enables OAuth API calls.
  When false, completely inert (no token read, no API call). Inert on Vertex.

## WebSocket API

Two endpoints run side by side during the v1→v2 rollout. Each has its own
`ConnectionManager`; the streams do not cross-broadcast. Pick one per
client.

### v1 — `/ws`

**Endpoint:** `ws://host:port/ws` — receive-only JSON text frames.

**Message types:** `session_discovered`, `session_ended`, `session_cleared`,
`hook_event`, `agent_started`, `agent_stopped`, `statusline_update`,
`limits_updated`. Every message has `type`, `platform`, `timestamp`. Most
have `session_id`.

**Bootstrap:** `GET /api/v1/sessions`.

### v2 — `/ws/v2`

**Endpoint:** `ws://host:port/ws/v2` — receive-only JSON text frames.

**Message types:** one per `claude_log_*` table:
- `hook_logged` — every row in `claude_log_hooks`. Carries the row's
  columns plus derived `process_id`, `epoch_id`, `log_id`. Clients
  dispatch on `event_name` (`SessionStart`, `PreToolUse`, `Stop`, etc.)
  for the typed signal they want.
- `statusline_logged` — every row in `claude_log_statuslines`. Cost,
  context, model, etc., plus derived ids and `log_id`.
- `pid_death_logged` — every row in `claude_log_pid_deaths`.
  Process-scoped (no session_id/epoch_id). Client action: mark every
  session under `process_id` as ended.
- `api_limits_logged` — every row in `claude_log_api_limits`. Account-
  scoped (no process_id/session_id). Carries `five_hour_utilization`,
  `five_hour_resets_at`, `seven_day_utilization`, `seven_day_resets_at`
  (resets normalized to epoch seconds) plus `log_id`. Other buckets
  (e.g. `seven_day_opus`) live in `raw_response` only — fetch via
  `GET /api/v2/log/api-limits` if you need them.

**`raw_payload` is not on the wire** for `hook_logged` /
`statusline_logged` — clients that need it fetch via `/api/v2/log/hooks`
or `/api/v2/log/statuslines` (raw_payload included by default on those
endpoints).

**Bootstrap:** `GET /api/v2/processes?active=true` for the initial process
list with nested sessions, then apply incoming frames.

**Full v2 protocol:** [docs/design/websocket.md](docs/design/websocket.md).

### Common to both

**Ping/pong:** AgentPulse does NOT send WebSocket pings. Clients that
expect server-initiated pings will timeout and disconnect. Configure the
client to either disable ping expectations or send its own keep-alive
pings.

**Client guides:**
- v1: [docs/old/clients.md](docs/old/clients.md)
- v2: [docs/design/clients.md](docs/design/clients.md)

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
