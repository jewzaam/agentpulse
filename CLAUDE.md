# AgentPulse

Backend service that collects Claude Code agent state and exposes it via REST + WebSocket.

## Commands

```bash
make check          # Run format, lint, typecheck, test, coverage (default)
make test           # Run pytest
make coverage       # Run pytest with 80% coverage threshold
make lint           # Lint with flake8
make typecheck      # Type check with mypy
make format         # Format with black
make run            # Start the service (DEBUG=1 for debug logging)
```

## Architecture

```
src/agentpulse/
├── app.py                        # FastAPI factory + lifespan (discovery loop)
├── config.py                     # Settings from env vars (AGENTPULSE_* prefix)
├── db.py                         # SQLite singleton (aiosqlite, WAL mode)
├── models.py                     # Normalized Pydantic response models
├── websocket.py                  # ConnectionManager + /ws endpoint
├── platforms/claude/
│   ├── hooks.py                  # POST /hooks/claude — hook event ingestion
│   ├── discovery.py              # Session file scanner + PID validation
│   ├── schema.py                 # claude_* table DDL + all CRUD queries
│   └── models.py                 # Claude-specific Pydantic models + derive_state()
└── api/
    ├── sessions.py               # Normalized endpoints: /api/v1/sessions, /summary
    └── claude.py                 # Raw endpoints: /api/v1/claude/sessions, /events
```

## Key Design Decisions

- **Raw events stored, state derived at query time** — DB has `last_event` and `last_tool`,
  not a `state` column. `derive_state()` in `platforms/claude/models.py` computes state
  strings at the API layer. Do not store computed states.
- **Platform-scoped tables** — All tables prefixed `claude_` (e.g., `claude_sessions`,
  `claude_events`). Future platforms get their own tables. Normalization happens in
  `api/sessions.py`, not in storage.
- **Coexists with claude-dashboard** — Runs on port 17385 (dashboard uses 17384).
- **Cost/usage stream is deferred** — `claude_costs` table exists but is not populated.
  `POST /costs/claude` returns 501.

## Testing

- `tests/conftest.py` has autouse safety guards: blocks subprocess, filesystem writes
  outside tmp_path, real HTTP, and isolates DB to tmp_path
- Each test file creates its own async `db` fixture via `init_db(db_path=tmp_path / "test.db")`
- Use `asyncio_mode = "auto"` — async test functions run automatically
- API tests use `FastAPI TestClient` with routers wired directly (no lifespan)

## Gotchas

- `db.py` uses a module-level singleton (`_db`). Tests must call `close_db()` in teardown
  to avoid leaking connections across test files.
- `discover_sessions()` is sync I/O wrapped in `asyncio.to_thread()` — it reads files
  from `~/.claude/sessions/`. Tests mock `validate_pid` to avoid psutil hitting real PIDs.
- The `src/` layout means Makefile targets use `SRC_DIR = src/$(PACKAGE_NAME)` for format,
  lint, typecheck — not the bare package name.
