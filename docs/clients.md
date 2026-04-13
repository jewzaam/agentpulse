# AgentPulse Client Guide

## Connecting

**REST API:** `http://host:port/api/v1/...`
**WebSocket:** `ws://host:port/ws`

Default host/port: `127.0.0.1:17385` (configured in `config.json`).

## Bootstrap Pattern

1. Connect to WebSocket
2. Fetch initial state: `GET /api/v1/sessions` (all sessions with current data)
3. Optionally fetch `GET /api/v1/limits` (usage quotas, if enabled)
4. Apply WebSocket messages incrementally to the local state

## WebSocket Protocol

- **Receive-only** — the server does not process incoming messages from clients
- **No server-initiated pings** — clients that expect pings will timeout and disconnect. Disable ping expectations or send client-side keep-alive
- **JSON text frames** — every message is `JSON.parse()`-able
- **All messages have:** `type` (string), `platform` (string), `timestamp` (float, Unix epoch)
- **Most messages have:** `session_id` (string, UUID)

## Message Types

### session_discovered

Fired when discovery finds a new session file in `~/.claude/sessions/`. This is the first time agentpulse sees the session — it may not have any hook or statusline data yet.

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

**Client action:** Add the session to local state. Fetch `GET /api/v1/sessions/{id}` for full details if needed.

### session_ended

Fired when a session's PID dies (detected by discovery) or a `SessionEnd` hook event arrives. The session remains in the database with `ended_at` set — it is not deleted.

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

**Client action:** Mark the session as ended in local state. Do not remove it — the user may want to see historical sessions.

### session_cleared

Fired when `POST /api/v1/sessions/{id}/clear` is called. Resets the session's last_event/last_tool/last_event_at to null and ends all active agents. Used when Claude Code is interrupted and no hook events fire.

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

**Client action:** Reset the session's state display (last event, tool, agents) to idle/empty.

### hook_event

Fired on every Claude Code hook event — tool calls, prompts, stops. This is the primary state change signal. The `event_name` tells you what happened.

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

Common `event_name` values:
- `UserPromptSubmit` — user sent a message
- `PreToolUse` — about to call a tool (`tool_name` has which one)
- `PostToolUse` — tool call completed
- `PermissionRequest` — waiting for user to approve a tool call
- `Stop` — Claude finished responding
- `Notification` — system notification

When `agent_id` is non-null, the event is from a subagent, not the main session.

**Client action:** Update the session's displayed state based on `event_name`. Use `derive_state` logic if you want to compute working/idle/permission_required: PreToolUse→working, PermissionRequest→permission_required, Stop→idle, etc.

### agent_started

Fired when a subagent starts within a session.

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

**Client action:** Add the agent to the session's agent list. Increment agent count.

### agent_stopped

Fired when a subagent completes or is cancelled.

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

**Client action:** Mark the agent as ended. Decrement active agent count.

### statusline_update

Fired when statusline data arrives from Claude Code (cost, tokens, context window, model). This is the richest data source for session metrics. Fires frequently — typically after each tool call.

```json
{
  "type": "statusline_update",
  "platform": "claude",
  "session_id": "uuid",
  "cost_usd": 5.25,
  "context_used_pct": 25,
  "model_name": "Opus 4.6 (1M context)",
  "total_input_tokens": 10000,
  "total_output_tokens": 50000,
  "lines_added": 100,
  "lines_removed": 20,
  "timestamp": 1776052424.367
}
```

**Client action:** Update the session's cost, context, model, and token displays. This is a full snapshot — replace, don't accumulate.

### limits_updated

Fired when fresh usage limits are fetched from the Anthropic OAuth API. Only fires when `fetch_limits` is enabled in config and the TTL has expired. Not session-specific.

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
    "five_hour": {"utilization": 11.0, "resets_at": "2026-04-13T07:00:00+00:00"},
    "seven_day": {"utilization": 8.0, "resets_at": "2026-04-19T13:00:00+00:00"}
  }
}
```

**Client action:** Update the global limits display. Use `refresh_after` to know when the next update will arrive.

## REST API Reference

### Sessions

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/sessions` | All sessions. `?active=true` for live only |
| GET | `/api/v1/sessions/{id}` | Session detail with agents |
| GET | `/api/v1/sessions/{id}/events` | Event history. `?limit=N&offset=N&since=T` |
| GET | `/api/v1/sessions/{id}/agents` | Session's agents |
| POST | `/api/v1/sessions/{id}/clear` | Reset session state |
| GET | `/api/v1/summary` | Aggregate stats + limits |

### Limits

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/limits` | Current usage limits |
| GET | `/api/v1/limits/history` | Historical snapshots. `?limit=N&offset=N&since=T` |

### Platform-Specific (Raw)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/claude/sessions` | Raw claude_sessions rows |
| GET | `/api/v1/claude/events` | Raw events. `?session_id=X&event_name=X&since=T&until=T` |

### Ingestion (Internal)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/hooks/claude` | Hook relay events |
| POST | `/statusline/claude` | Statusline data |

## Derived State

The REST API includes a `derived_state` field on sessions. This is computed from `last_event` and `last_tool` at query time. WebSocket clients that want to derive state from `hook_event` messages can use this mapping:

| event_name | tool_name | derived_state |
|------------|-----------|---------------|
| `PreToolUse` | `AskUserQuestion` | `awaiting_input` |
| `PreToolUse` | (any other) | `working` |
| `PermissionRequest` | `AskUserQuestion` | `awaiting_input` |
| `PermissionRequest` | (any other) | `permission_required` |
| `UserPromptSubmit` | — | `working` |
| `PostToolUse` | — | `working` |
| `Stop` | — | `idle` |

When a session has active agents, the effective state is the highest-priority state across the main session and all agents. Priority (highest first): `permission_required` > `awaiting_input` > `working` > `ready` > `idle`.
