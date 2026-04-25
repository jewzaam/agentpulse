# WebSocket Protocol

Companion to [`api.md`](api.md). What clients receive in real time.
Aligned with the data model in [`database.md`](database.md).

## Principles

1. **One message per log row.** Every broadcast corresponds to a
   row that landed in a `claude_log_*` table. No higher-level
   "state mutated" broadcasts that don't map to a log write.
2. **Carry the natural identity on every message.** A consumer that
   missed earlier messages can route a single message correctly
   using only its content — no REST lookup required.
3. **One time unit.** All timestamps are Unix epoch seconds (float).
4. **Server-derived ids on every message.** `process_id` and
   `epoch_id` ship with each message so clients can mirror the REST
   projection without recomputing.
5. **No silent skips.** Every log write broadcasts. There is no
   "deferred" insert.

## Endpoint

`ws://{host}:{port}/ws/v2`. Default `127.0.0.1:17385/ws/v2`.

## Connection semantics

- **Receive-only.** The server reads to detect disconnect; client
  → server frames are discarded.
- **No server pings.** Configure ping expectations off, or send
  client-side keep-alives.
- **JSON text frames.** UTF-8, `JSON.parse()`-able.
- **No replay.** Bootstrap via REST after connect/reconnect.
- **No subscription filtering.** Every connection sees every
  broadcast. Clients filter by `type`, `session_id`, `process_id`.

## Common envelope

| Field | Type | Notes |
|---|---|---|
| `type`         | string | One of the values listed below. |
| `platform`     | string | `"claude"` today. |
| `timestamp`    | float seconds | Server wall clock at broadcast. Roughly equals the row's `received_at` / `observed_at`. |

## Process / session / epoch identity

Every **process-scoped** message also has:

| Field | Type | Notes |
|---|---|---|
| `process_id`    | string | Server-derived (see `api.md`). |
| `pid`           | int    | Claude Code's PID at event time. |
| `source_system` | string | Hostname of the machine running Claude. |
| `cwd`           | string | Current working directory at event time. |

Every **session-scoped** message adds:

| Field | Type | Notes |
|---|---|---|
| `session_id`    | string | The session the message belongs to. |
| `epoch_id`      | string | The current epoch the session is in. |

Every **agent-scoped** message adds:

| Field | Type | Notes |
|---|---|---|
| `agent_id`      | string | The agent the message is about. |

Process messages include process keys but not session ids.
Session messages include process + session keys.
Agent messages include process + session + agent keys.

## Message catalog

| `type` | Source | Scope |
|---|---|---|
| `hook_logged`      | one row in `claude_log_hooks` | session |
| `statusline_logged`| one row in `claude_log_statuslines` | session |
| `pid_death_logged` | one row in `claude_log_pid_deaths` | process |
| `api_limits_logged`| one row in `claude_log_api_limits` | global |

A single `hook_logged` carries every observable session event;
clients dispatch on `event_name` (`SessionStart`, `SessionEnd`,
`UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`,
`Stop`, `Notification`, `SubagentStart`, `SubagentStop`,
`clear_state`, …) for the typed signal they want.

Aligning the WebSocket protocol with log writes makes the
server-side dispatch a one-line "broadcast the row I just inserted"
function and keeps the data model and the wire surface visibly the
same.

## Message shapes

### `hook_logged`

One row in `claude_log_hooks`. Mirrors the row's columns one-to-one,
plus derived ids and `type`/`platform`/`timestamp` envelope.

```json
{
  "type": "hook_logged",
  "platform": "claude",
  "timestamp": 1776947339.99,
  "process_id": "a1b2c3d4e5f60718",
  "epoch_id": "9f8e7d6c5b4a3210",
  "pid": 68718,
  "source_system": "host.example",
  "cwd": "/path/to/project",
  "session_id": "a415cfed-95a4-...",
  "agent_id": null,
  "event_name": "PreToolUse",
  "tool_name": "Bash",
  "agent_type": null,
  "permission_mode": "acceptEdits",
  "entrypoint": "vscode",
  "claude_version": "2.1.117",
  "received_at": 1776947339.99,
  "received_by": "hook-endpoint",
  "log_id": 4123
}
```

`log_id` is the row's `id` in `claude_log_hooks` — useful for clients
that page raw history via `/api/v2/log/hooks` and want to dedupe.
`raw_payload` is **not** included in WebSocket frames; clients that
need it fetch via `/api/v2/log/hooks?id=...` or
`/api/v2/sessions/{id}/events?include=raw`. Keeping payloads off the
wire bounds the broadcast size; raw access is one REST hop away.

#### Common `event_name` values and what they mean

| `event_name`        | What the client typically does |
|---|---|
| `SessionStart`      | New session — render it under its process. |
| `SessionEnd`        | Session ended — mark it terminal but keep history. |
| `UserPromptSubmit`  | New turn started. |
| `PreToolUse`        | Tool about to run. `tool_name` says which. |
| `PostToolUse`       | Tool completed. |
| `PermissionRequest` | Awaiting human approval for a tool call. |
| `Stop`              | Turn ended. |
| `Notification`      | Show the message somewhere. |
| `SubagentStart`     | Add the agent to the session's agent list. |
| `SubagentStop`      | Mark the agent as ended. |
| `clear_state`       | Synthetic, written by `POST /api/v2/sessions/{id}/clear`. The frame carries the same `pid` / `source_system` / `cwd` / `process_id` / `epoch_id` as a real hook (copied from the session's most recent prior hook). Reset the session display. |

The list is non-exhaustive — Claude Code adds events. Clients should
ignore unknown `event_name` values rather than error.

### `statusline_logged`

One row in `claude_log_statuslines`. Mirrors columns; same shape as
the `StatuslineResponse` REST projection (`api.md`) plus envelope.

```json
{
  "type": "statusline_logged",
  "platform": "claude",
  "timestamp": 1776947339.99,
  "process_id": "a1b2c3d4e5f60718",
  "epoch_id": "9f8e7d6c5b4a3210",
  "pid": 68718,
  "source_system": "host.example",
  "cwd": "/path/to/project",
  "project_dir": "/path/to/project",
  "session_id": "a415cfed-95a4-...",
  "model_id": "claude-opus-4-7[1m]",
  "model_name": "Opus 4.7 (1M context)",
  "claude_version": "2.1.117",
  "cost_usd": 83.58,
  "total_input_tokens": 292478,
  "total_output_tokens": 512427,
  "context_used_pct": 6,
  "context_window_size": 1000000,
  "cache_read_tokens": 9000,
  "cache_creation_tokens": 500,
  "duration_seconds": 120.0,
  "api_duration_seconds": 30.0,
  "lines_added": 6269,
  "lines_removed": 944,
  "exceeds_200k_tokens": false,
  "received_at": 1776947339.99,
  "received_by": "statusline-endpoint",
  "log_id": 891
}
```

Important behaviors:

- **`cost_usd` is process-cumulative**, not session-cumulative.
  Persists across `/clear` within one process, resets on
  `claude --resume` into a new process.
- **No `cost_by_day` field on this message.** Daily breakdown is a
  derived view available at `GET /api/v2/processes/{process_id}`
  (in the process body). Clients that need daily totals refetch on
  a coarse cadence; the per-statusline broadcast stays small.
- **Always broadcast.** A statusline whose `session_id` doesn't yet
  match any hook is still a valid log row and produces a broadcast.
  Consumers tracking sessions by `session_id` may receive a
  `statusline_logged` before the first `hook_logged` for that
  session — they should create a pending session entry on first
  sight.

### `pid_death_logged`

One row in `claude_log_pid_deaths`. Process-scoped, not
session-scoped — a single death affects every session in the
process.

```json
{
  "type": "pid_death_logged",
  "platform": "claude",
  "timestamp": 1776948000.42,
  "process_id": "a1b2c3d4e5f60718",
  "pid": 68718,
  "source_system": "host.example",
  "cwd": "/path/to/project",
  "observed_at": 1776948000.42,
  "observed_by": "pid-watcher",
  "log_id": 14
}
```

Client action: mark every session belonging to `process_id` as
ended, mark every active agent within those sessions as ended, set
the process's `pid_alive` to false. The corresponding `ended_at`
values can be derived as `observed_at` for any open epoch in the
process.

### `api_limits_logged`

One row in `claude_log_api_limits`. Global (not session-scoped, not
process-scoped, no machine scope — see the **Single Anthropic
account** constraint in `database.md`).

```json
{
  "type": "api_limits_logged",
  "platform": "claude",
  "timestamp": 1776052424.367,
  "received_at": 1776052424.367,
  "received_by": "limits-fetcher",
  "log_id": 27,
  "five_hour_utilization": 11.0,
  "five_hour_resets_at": 1776920400.0,
  "seven_day_utilization": 8.0,
  "seven_day_resets_at": 1776981600.0
}
```

`*_resets_at` values are seconds. The full upstream response is
available at `GET /api/v2/limits/history` with `include=raw` when a
client wants the buckets we don't promote (e.g. `seven_day_opus`).

This is the only message type with no `process_id` — limits are
account-scoped, not process-scoped.

## Bootstrap pattern

1. Connect to `/ws/v2`.
2. Buffer incoming frames.
3. `GET /api/v2/processes?active=true` — initial process list with
   nested sessions.
4. `GET /api/v2/limits` if interested.
5. Apply buffered + subsequent broadcasts.

## Reconnect pattern

The server provides no replay. **Reconnect by re-running bootstrap.**
A fresh process list is a complete snapshot of the current state;
gap-filling from raw log tables is unnecessary for live clients.

`log_id` on each frame is still useful — it lets clients that page
raw history (`GET /api/v2/log/{table}`) dedupe rows, and it gives
clients a stable cursor when investigating drift. It is **not** a
required input to the live reconnect path.

