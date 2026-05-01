# Python Client Library

The `agentpulse[client]` extra ships a ready-made Python client so
consumers don't hand-roll the streaming loop, bootstrap, frame
dispatch, and reconnect logic. If you're writing a Python consumer,
use this library — `clients.md` (the wire protocol guide) is for
non-Python consumers and library maintainers.

## Install

```bash
pip install "agentpulse[client]"
```

The extra brings in `httpx` and `websockets`. The server-side
dependencies (`fastapi`, `aiosqlite`, etc.) come along for the ride
when you install the full `agentpulse` package; install only the
client extra if you don't run the server.

## Two clients, one core

| Class | Use when |
|---|---|
| `agentpulse.client.AsyncAgentPulseClient` | You already run an asyncio loop (FastAPI, aiohttp, your own). |
| `agentpulse.client.AgentPulseClient` | You're writing a sync app with its own main loop (Tk, Qt, CLI tool). |

The sync class wraps the async one in a daemon thread. Snapshot reads
are thread-safe on both. Choose whichever matches your runtime.

## Sync usage (Tk example)

```python
from agentpulse.client import AgentPulseClient, load_client_config

config = load_client_config()
if config is None:
    raise RuntimeError("AgentPulse not configured")

# Marshaler routes callbacks onto the Tk main thread. Without it,
# callbacks fire on the asyncio worker thread and Tk widget updates
# from there will throw TclError.
client = AgentPulseClient(config, marshaler=lambda f: root.after(0, f))

def render() -> None:
    sessions = client.sessions()
    state = client.connection_state()
    # update widgets...

client.on_change(render)
client.start()           # blocks until first bootstrap completes
root.mainloop()
client.stop()
```

For Qt, use `QTimer.singleShot(0, f)` or a signal-emit as the
marshaler. For headless apps that handle threading themselves, omit
`marshaler` — callbacks fire on the asyncio worker thread.

## Async usage

```python
import asyncio
from agentpulse.client import AsyncAgentPulseClient, load_client_config

async def main() -> None:
    config = load_client_config()
    assert config is not None
    client = AsyncAgentPulseClient(config)

    def on_change() -> None:
        for s in client.sessions():
            print(s.session_id, s.derived_state, s.total_cost_usd)

    client.on_change(on_change)
    await client.start()
    try:
        await asyncio.Event().wait()  # run forever
    finally:
        await client.stop()

asyncio.run(main())
```

## Snapshot API

| Method | Returns | Notes |
|---|---|---|
| `client.sessions()` | `tuple[Session, ...]` | All known sessions, today + active. |
| `client.session(session_id)` | `Session \| None` | Lookup by id. |
| `client.limits()` | `Limits \| None` | `None` when `fetch_limits=False` or never seen. |
| `client.connection_state()` | `ConnectionState` | `DISCONNECTED`, `CONNECTING`, `CONNECTED`. |

`Session`, `Limits`, `Epoch`, `Agent` are frozen dataclasses; field
names mirror the v2 wire format verbatim (see `api.md`).

`Session` includes statusline-derived counters (`total_input_tokens`,
`total_output_tokens`, `lines_added`, `lines_removed`) that aren't on
`SessionResponse`. They default to 0 and populate from
`statusline_logged` frames or the bootstrap statuslines fetch.

## Callbacks

| Hook | Fires |
|---|---|
| `on_change(fn)` | `fn()` after every store mutation. |
| `on_session_added(fn)` | `fn(session)` on first sight of a `session_id`. |
| `on_session_ended(fn)` | `fn(session)` when `ended_at` transitions from `None`. |
| `on_process_ended(fn)` | `fn(process_id)` after `pid_death_logged` fan-out. |
| `on_limits_changed(fn)` | `fn(limits)` after `api_limits_logged` or REST refresh. |
| `on_connection_state(fn)` | `fn(state)` on every state change. |

All callbacks are sync (`def fn(...)`). They run on the mutator thread
under the store lock — **do not block them**. Push slow work onto a
queue or `asyncio.create_task` and return immediately.

The sync facade's `marshaler` redirects callback dispatch onto your UI
thread before invoking the registered function, so blocking your UI
thread inside a callback still freezes the UI — same rule applies.

## Bootstrap and reconnect

`start()` opens the WebSocket, runs bootstrap, and returns once the
first snapshot is loaded (or the first attempt failed). Bootstrap is
two REST calls today:

1. `GET /api/v2/sessions?since=<midnight_local>`
2. `GET /api/v2/log/statuslines?since=<midnight_local>&order=desc&limit=1000`
   (seeds tokens / lines / cost — fields not on `SessionResponse`)
3. `GET /api/v2/log/api-limits?order=desc&limit=1` if `fetch_limits=True`

On disconnect, the store is **not** cleared. Last-known data
persists; `connection_state()` returns `DISCONNECTED`. Consumers
decide whether to dim the UI, show a banner, etc. On reconnect the
client transitions `DISCONNECTED → CONNECTING`, re-runs bootstrap,
then `→ CONNECTED`.

Reconnect uses exponential backoff capped at 30s; tests can override
via `ClientConfig.initial_backoff_s` / `backoff_cap_s`.

## Manual refresh

```python
await client.refresh()                        # full re-bootstrap
await client.refresh_session("abc-123")       # one session by id
```

Use `refresh()` at local midnight if you display per-day costs (today's
bucket rolls over). The library does **not** schedule this for you —
hook it to your existing scheduler (Tk `after`, asyncio task, etc.).

## Producer-side helpers

For SDK consumers that need to forward statuslines or hooks to
AgentPulse:

```python
await client.post_statusline(payload)         # POST /statusline/claude
await client.post_hook(payload)               # POST /hooks/claude
```

Both return `bool` (True on 2xx). Network failures return False, never
raise. The payload shape matches the `/statusline/claude` and
`/hooks/claude` endpoints — see `api.md`.

For one-shot POSTs without a streaming client, use the module-level
helpers:

```python
import httpx
from agentpulse.client import rest

async with httpx.AsyncClient() as http:
    await rest.post_statusline(http, "127.0.0.1", 17385, payload)
```

## Configuration

`load_client_config()` reads `~/.claude/agentpulse/config.json` and
returns a `ClientConfig` (or `None` if the file is missing/malformed).
Only client-relevant keys are read:

| Key | Default | Notes |
|---|---|---|
| `host` | required | |
| `port` | required | |
| `fetch_limits` | `True` | When False, skips limits bootstrap and ignores `api_limits_logged` frames. |

Other knobs are construction-time only:

```python
ClientConfig(
    host="127.0.0.1",
    port=17385,
    fetch_limits=True,
    initial_backoff_s=1.0,
    backoff_cap_s=30.0,
    http_timeout_s=5.0,
)
```

## Common pitfalls

- **Blocking inside a callback** — freezes other writers (sync facade)
  or stalls the asyncio loop. Push work to a queue.
- **Forgetting the marshaler in Tk/Qt apps** — Tcl/Qt errors appear
  far from the cause. Pass a marshaler if your callbacks touch widgets.
- **Treating disconnect as data loss** — the store is preserved across
  disconnects. Use `connection_state()` to advertise staleness, not
  empty `sessions()`.
- **Calling `post_*` from inside a callback** — works, but you've
  blocked the store lock until the network round-trip returns. Use
  the async client directly or queue the post.
- **Expecting `sessions()` to include yesterday** — bootstrap is
  today-only. For history, use the REST log endpoints directly via
  `agentpulse.client.rest`.

## Wire protocol

If you're porting to another language or extending the library, see:

- [`clients.md`](clients.md) — wire protocol guide for client authors.
- [`api.md`](api.md) — full REST surface.
- [`websocket.md`](websocket.md) — frame shapes and dispatch rules.
