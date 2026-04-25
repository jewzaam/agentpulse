# WebSocket Protocol — Current (v1)

What the running service broadcasts today. Sourced from
[`src/agentpulse/events.py`](../../src/agentpulse/events.py),
[`src/agentpulse/websocket.py`](../../src/agentpulse/websocket.py),
and the dispatch sites in `platforms/claude/hooks.py`. Pinned here so
the v2 design has a fixed reference.

## Endpoint

`ws://{host}:{port}/ws` — single global stream. Default
`127.0.0.1:17385/ws`.

## Connection semantics

- **Receive-only.** The server reads incoming frames only to detect
  disconnects; client → server messages are silently discarded.
- **No server-initiated pings.** Clients that expect server pings will
  timeout. Disable the expectation, or send client-side keep-alives.
- **JSON text frames.** Every broadcast is a `JSON.parse()`-able UTF-8
  string.
- **No backpressure or replay.** A slow consumer that fails to drain
  is marked dead and dropped. Reconnecting clients miss messages
  emitted while disconnected — bootstrap via REST after reconnect.

## Time units

`timestamp` on every message is **Unix epoch seconds (float)**.
Limits payloads nested under `limits_updated` carry their own
`fetched_at` / `refresh_after` (also seconds) plus ISO-8601
`resets_at` strings inherited from the upstream OAuth API.

## Common envelope

Every message has:

| Field | Type | Notes |
|---|---|---|
| `type` | string | One of the values listed below. |
| `platform` | string | Always `"claude"` today. |
| `timestamp` | float | Unix epoch seconds. Server wall clock at broadcast. |

Every **session-scoped** message also has:

| Field | Type | Notes |
|---|---|---|
| `session_id` | string | The session the message belongs to. |
| `cwd` | string \| null | Current working directory at event time, when known. |
| `event_name` | string \| null | Hook event, when applicable. |
| `tool_name` | string \| null | Tool involved, when applicable. |
| `agent_id` | string \| null | Subagent id, when from a subagent. |

**Note:** session-scoped messages do **not** carry `pid` or
`source_system`. A client that needs the enclosing process keys must
look up the session via REST. See "v1 warts" at the bottom.

## Message catalog

| `type` | Trigger | Session-scoped |
|---|---|---|
| `session_discovered` | First time a `session_id` is seen via hook event, or a previously-ended session resumes. | yes |
| `session_ended`      | `SessionEnd` hook fires, or discovery loop observes `pid_alive` flip to false. | yes |
| `session_cleared`    | `POST /api/v1/sessions/{id}/clear` is called. | yes |
| `hook_event`         | Any hook event other than the ones promoted to dedicated types below. | yes |
| `agent_started`      | `SubagentStart` hook event. | yes |
| `agent_stopped`      | `SubagentStop` hook event. | yes |
| `statusline_update`  | `POST /statusline/claude` for a known session. Skipped when the session is unknown ("deferred"). | yes |
| `limits_updated`     | Limits-fetch loop pulls a fresh OAuth API snapshot. | no |

The hook event-name → message-type mapping is in
[`events.py`](../../src/agentpulse/events.py) (`broadcast_hook_event`):
`SessionEnd → session_ended`, `SubagentStart → agent_started`,
`SubagentStop → agent_stopped`, anything else → `hook_event`.

## Message shapes

### `session_discovered`

```json
{
  "type": "session_discovered",
  "platform": "claude",
  "session_id": "uuid",
  "event_name": null,
  "tool_name": null,
  "agent_id": null,
  "cwd": "/path/to/project",
  "timestamp": 1776052424.367
}
```

Add the session to the local view; optionally fetch
`GET /api/v1/sessions/{id}` for full session detail.

### `session_ended`

```json
{
  "type": "session_ended",
  "platform": "claude",
  "session_id": "uuid",
  "event_name": null,
  "tool_name": null,
  "agent_id": null,
  "cwd": "/path/to/project",
  "timestamp": 1776052424.367
}
```

Note: `event_name` is null when the trigger was a discovery-loop PID
death; the `SessionEnd` hook path uses `broadcast_hook_event` which
sets `event_name` to `"SessionEnd"`. Both produce a `session_ended`
message; the field difference is observable. Mark the session ended;
do not delete (history is kept).

### `session_cleared`

```json
{
  "type": "session_cleared",
  "platform": "claude",
  "session_id": "uuid",
  "event_name": "clear_state",
  "tool_name": null,
  "agent_id": null,
  "cwd": "/path/to/project",
  "timestamp": 1776052424.367
}
```

Reset the session's displayed state (last event, tool, agents) to
idle/empty. Triggered only by `POST /api/v1/sessions/{id}/clear` —
which is the only way to recover when the user interrupted Claude
mid-turn (no hook fires for an interrupt; see the runtime
reference).

### `hook_event`

```json
{
  "type": "hook_event",
  "platform": "claude",
  "session_id": "uuid",
  "event_name": "PreToolUse",
  "tool_name": "Bash",
  "agent_id": null,
  "cwd": "/path/to/project",
  "timestamp": 1776052424.367
}
```

Common `event_name` values seen:
`UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`,
`Notification`, `Stop`. When `agent_id` is non-null the event came
from a subagent.

Update the session's derived state from `event_name` / `tool_name`
(mapping in `docs/old/api.md`).

### `agent_started`

```json
{
  "type": "agent_started",
  "platform": "claude",
  "session_id": "uuid",
  "event_name": "SubagentStart",
  "tool_name": null,
  "agent_id": "agent-uuid",
  "cwd": "/path/to/project",
  "timestamp": 1776052424.367
}
```

### `agent_stopped`

```json
{
  "type": "agent_stopped",
  "platform": "claude",
  "session_id": "uuid",
  "event_name": "SubagentStop",
  "tool_name": null,
  "agent_id": "agent-uuid",
  "cwd": "/path/to/project",
  "timestamp": 1776052424.367
}
```

### `statusline_update`

```json
{
  "type": "statusline_update",
  "platform": "claude",
  "session_id": "uuid",
  "cost_usd": 5.25,
  "prior_cost_usd": 4.15,
  "context_used_pct": 25,
  "model_name": "Opus 4.6 (1M context)",
  "total_input_tokens": 10000,
  "total_output_tokens": 50000,
  "lines_added": 100,
  "lines_removed": 20,
  "timestamp": 1776052424.367
}
```

`cost_usd` is the **process-cumulative** cost as reported by Claude's
statusline. `prior_cost_usd` is the max statusline `cost_usd`
recorded before local midnight for this session_id — used by clients
to derive "today's cost" as `cost_usd - prior_cost_usd`. Tokens and
lines are also process-cumulative. `context_used_pct` and
`model_name` are per-session.

Replace (do not accumulate) the displayed cost / tokens / lines /
context.

Statusline rows whose `session_id` is not yet known to
`claude_sessions` are stored but **not broadcast** — the endpoint
returns `{"status":"deferred"}` and the next hook for that session
will create the row.

### `limits_updated`

```json
{
  "type": "limits_updated",
  "platform": "claude",
  "timestamp": 1776052424.367,
  "limits": {
    "available": true,
    "fetched_at": 1776052424.367,
    "age_seconds": 0.0,
    "stale": false,
    "refresh_after": 1776052544.367,
    "five_hour": {
      "utilization": 11.0,
      "resets_at": "2026-04-13T07:00:00+00:00"
    },
    "seven_day": {
      "utilization": 8.0,
      "resets_at": "2026-04-19T13:00:00+00:00"
    }
  }
}
```

Not session-scoped — single global limits view. Display the latest
snapshot and use `refresh_after` to know when the next update will
arrive.

## Bootstrap pattern

1. Connect to `/ws`.
2. Buffer incoming messages.
3. Fetch `GET /api/v1/sessions` for the initial state.
4. Optionally fetch `GET /api/v1/limits`.
5. Apply buffered and subsequent WebSocket messages incrementally.

There is no server-side replay. A client that disconnects and
reconnects must re-bootstrap.

## Known issues / v1 warts

- **No `pid` / `source_system` on session-scoped messages.** Clients
  that want to group sessions by their enclosing process must look
  up each session via REST first.
- **Two trigger paths for `session_ended` produce slightly different
  payloads.** SessionEnd hook → `event_name: "SessionEnd"`; PID death
  via discovery → `event_name: null`. Clients should treat both as
  "session ended" and not depend on `event_name`.
- **Statusline deferral is invisible to clients.** A statusline POST
  for an unknown session produces no broadcast. Clients learn about
  the data only later, when a hook creates the session and subsequent
  REST queries pick up the pre-existing rows.
- **PID death is per-session.** When a process hosting N sessions
  dies, the discovery loop emits N independent `session_ended`
  messages, one per session. There is no aggregate signal for
  "process gone."
- **No event for limits being unavailable.** `limits_updated` only
  fires on successful fetches; failures are logged server-side and
  stay out of the wire.
- **No subscription filtering.** Every connection receives every
  broadcast; clients filter on `type` and `session_id` themselves.
