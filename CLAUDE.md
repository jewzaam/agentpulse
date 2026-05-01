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

Key settings: `host`, `port`, `db_path`, `log_file`, `pidfile_dir`, `log_level`,
`discovery_interval_seconds`, `fetch_limits` (bool — enables OAuth API usage limit
fetching), `broadcast_debounce_ms` (int, default 500 — hook broadcast debounce
window; 0 disables throttling). `pidfile_dir` defaults to `~/.claude/agentpulse/`; override it to run a
second instance side-by-side (e.g. `oneshot/` for testing) without colliding on
the live deployment's pidfile.

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
own processing. AgentPulse stores each update in `claude_log_statuslines` (append-only
log with extracted numerics + raw payload).

## Architecture

```
src/agentpulse/
├── app.py                        # FastAPI factory + lifespan (pid watcher + limits fetch loops)
├── config.py                     # Settings from config file (--config required)
├── db.py                         # SQLite singleton (aiosqlite, WAL mode)
├── platforms/claude/
│   ├── hooks.py                  # POST /hooks/claude, /statusline/claude
│   ├── entrypoint.py             # detect_entrypoint via psutil ancestor walk
│   ├── pid_watcher.py            # PID death watcher → claude_log_pid_deaths
│   ├── limits.py                 # OAuth API usage limit fetching + v2 broadcast
│   ├── schema.py                 # claude_log_* DDL + inserters
│   └── models.py                 # ClaudeHookPayload Pydantic model
├── api/
│   └── v2/
│       ├── router.py             # /api/v2/processes, /sessions, /log/{hooks,statuslines,pid-deaths,api-limits}
│       ├── models.py             # ProcessResponse, SessionResponse, EpochResponse, ...
│       ├── ids.py                # process_id, epoch_id (sha256 prefix-16)
│       ├── state.py              # derive_state (Stop → ready) + compute_effective_state
│       ├── throttle.py           # HookBroadcastThrottle — debounces same-state hook_logged broadcasts
│       ├── websocket.py          # /ws/v2 endpoint
│       ├── events.py             # broadcast_hook_logged, statusline_logged, pid_death_logged, api_limits_logged
│       └── queries/              # projection layer over claude_log_* tables
│           ├── log.py            # raw filtered reads
│           ├── enrich.py         # IdEnricher (per-request id cache)
│           ├── sessions.py       # session/epoch derivation
│           ├── processes.py      # process derivation (consults pid_deaths for ended_at/pid_alive)
│           └── _helpers.py       # shared latest-hook + agent helpers
└── client/                       # consumer library — `agentpulse[client]` extra
    ├── config.py                 # ClientConfig + load_client_config (~/.claude/agentpulse/config.json)
    ├── types.py                  # frozen dataclasses: Session, Epoch, Agent, Limits, ConnectionState
    ├── _state.py                 # ClientStore — frame application, RLock-guarded, callback dispatch
    ├── rest.py                   # async REST helpers (post_*, fetch_*); never raise on network failure
    ├── aio.py                    # AsyncAgentPulseClient — async core
    └── sync.py                   # AgentPulseClient — daemon-thread facade with marshaler
```

## Key Design Decisions

- **Raw events stored, state derived at query time** — `claude_log_*` tables are
  append-only. `derive_state()` in `api/v2/state.py` computes state strings at
  the API layer. Do not store computed states.
- **Platform-scoped tables** — All tables prefixed `claude_`:
  `claude_log_hooks`, `claude_log_statuslines`, `claude_log_pid_deaths`,
  `claude_log_api_limits`. Append-only logs; derived projections (process,
  session, epoch, agent) live in `api/v2/queries/`. Future platforms get
  their own tables.
- **Inserters return the row id** — `insert_log_hook`, `insert_log_statusline`
  return `int | None` (None when payload lacks `pid`). `insert_log_pid_death`
  returns `int` (always inserts). `insert_log_api_limits` returns
  `(int, dict)` — the row id plus the normalized fields the broadcast
  needs. Callers use the returned id to broadcast `*_logged` frames on
  `/ws/v2` after the commit.
- **REST responses and WebSocket broadcasts must match** — every data field on a
  session or event in the REST API must also appear in the corresponding WebSocket
  broadcast. Consumers should be able to build the same view from either source.
- **Config file is the only configuration source** — no env vars. Both the service
  (`--config`) and the hook relay (`--config`) read the same file.
- **cost_usd is session-cumulative** — Claude Code's `total_cost_usd` in
  statusline payloads never resets for a given `session_id`, even across
  `--resume` (new PID). The latest statusline row always holds the true
  session total. Session cost = `MAX(cost_usd)` across all rows. Process
  cost = `MAX - baseline`, where baseline is the session's cost before
  that process started. Do not SUM per-instance MAXes.
- **Limits fetching is optional** — `fetch_limits: true` enables OAuth API calls.
  When false, completely inert (no token read, no API call). Inert on Vertex.
- **hook_logged broadcasts are debounced** — `HookBroadcastThrottle` in
  `api/v2/throttle.py` debounces per `(session_id, agent_id)`. State changes
  and lifecycle events broadcast immediately; same-state events coalesce within
  a configurable window (`broadcast_debounce_ms`, default 500). Storage is
  unaffected — every hook still lands in `claude_log_hooks`. `derived_state`
  is computed server-side and included on every `hook_logged` frame.
- **Python consumers use `agentpulse.client`, not raw REST/WS** —
  `src/agentpulse/client/` ships a sync facade (`AgentPulseClient` with
  optional Tk/Qt marshaler) and an async core (`AsyncAgentPulseClient`)
  that already handle bootstrap, reconnect with exponential backoff,
  frame dispatch, cumulative-vs-replace cost semantics, and pid_death
  fan-out. New Python consumers should depend on `agentpulse[client]`
  rather than re-implementing the streaming loop. The wire-protocol
  guide at `docs/design/clients.md` is for non-Python consumers and
  library maintainers. See `docs/design/python-client.md` for usage.

## WebSocket API

**Endpoint:** `ws://host:port/ws/v2` — receive-only JSON text frames.

**Message types:** one per `claude_log_*` table:
- `hook_logged` — carries the row's columns plus derived `process_id`,
  `epoch_id`, `log_id`, and `derived_state`. Same-state events are
  debounced per `(session_id, agent_id)`; state changes and lifecycle
  events broadcast immediately. Clients dispatch on `event_name`
  (`SessionStart`, `PreToolUse`, `Stop`, etc.) for the typed signal
  they want.
- `statusline_logged` — every row in `claude_log_statuslines`. Cost,
  context, model, etc., plus derived ids and `log_id`. Includes
  `session_total_cost_usd` (server-derived global MAX(cost_usd) for
  the session, including this row) and `session_today_cost_usd` (the
  today delta in server local time) so clients can update per-session
  totals live. The full `cost_by_day` map is **not** on the wire —
  fetch from REST on a coarse cadence.
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

**Full protocol:** [docs/design/websocket.md](docs/design/websocket.md).

**Ping/pong:** AgentPulse does NOT send WebSocket pings. Clients that
expect server-initiated pings will timeout and disconnect. Configure the
client to either disable ping expectations or send its own keep-alive
pings.

**Client guide:** [docs/design/clients.md](docs/design/clients.md).

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
- `detect_entrypoint()` is sync and wrapped in `asyncio.to_thread()`. Tests
  mock psutil to avoid hitting real PIDs.
- The `src/` layout means Makefile targets use `SRC_DIR = src/$(PACKAGE_NAME)` for format,
  lint, typecheck — not the bare package name.
- Windows `ProactorEventLoop` raises `ConnectionResetError` when hook relay clients
  disconnect early. Suppressed by a logging filter in `app.py` on the `asyncio` logger.
- `aiosqlite` debug logging is suppressed (set to WARNING) to keep logs readable.
- Schema changes require dropping the DB if tables already exist — no migration system yet.
  `CREATE TABLE IF NOT EXISTS` won't add new columns to existing tables.
