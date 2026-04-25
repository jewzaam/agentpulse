# REST API — Current (v1)

What the running service exposes today. Sourced from
[`src/agentpulse/api/sessions.py`](../../src/agentpulse/api/sessions.py),
[`src/agentpulse/api/claude.py`](../../src/agentpulse/api/claude.py),
[`src/agentpulse/platforms/claude/hooks.py`](../../src/agentpulse/platforms/claude/hooks.py),
and [`src/agentpulse/models.py`](../../src/agentpulse/models.py).

Pinned here so the v2 design has a fixed reference. This file is **not**
updated as code evolves; once v2 lands, it represents the frozen v1
surface.

## Base URL

`http://{host}:{port}` — defaults `127.0.0.1:17385`, configured via
`~/.claude/agentpulse/config.json` (`host`, `port`).

## Versioning

All normalized endpoints are namespaced under `/api/v1/`. Platform-raw
endpoints are namespaced under `/api/v1/claude/`. Ingestion endpoints
(`/hooks/claude`, `/statusline/claude`, `/costs/claude`) are not
versioned — they are internal contracts between the relay scripts and
the service.

## Time units

Most fields are **Unix epoch seconds (float)**. The exception:
`started_at` on sessions is **milliseconds (int)** — historical
artifact of how Claude Code reports session start. Mixed units in one
response are a known wart of v1.

## Conceptual model

See [`docs/reference/claude-runtime.md`](../reference/claude-runtime.md)
for upstream Claude Code runtime semantics (process / session /
`/clear` / resume).

What's specific to v1's API surface: **sessions are the primary
entity**. They are returned as a flat list. Each session row carries
its own `pid`, `cwd`, `entrypoint`, `source_system`, `branch`, and the
process-cumulative cost / token / line counters. There is no separate
"process" entity; multiple sessions belonging to the same process
appear as independent rows in the list. Clients that want a per-process
view group rows by `(pid, source_system)` themselves.

## Endpoint catalog

### Normalized (cross-platform)

| Method | Path | Purpose |
|---|---|---|
| GET    | `/api/v1/sessions` | Flat list of sessions. `?active=true` filters to live sessions (`pid_alive=true`, `ended_at` null). |
| GET    | `/api/v1/sessions/{session_id}` | Session detail with embedded agents. 404 if unknown. |
| GET    | `/api/v1/sessions/{session_id}/events` | Event history. `?limit=N&offset=N&since=T`. |
| GET    | `/api/v1/sessions/{session_id}/agents` | Agents for a session. |
| POST   | `/api/v1/sessions/{session_id}/clear` | Reset session state (synthetic `clear_state` event, end active agents). Used when Claude is interrupted with no events firing — see the runtime reference for why interrupts produce no hook. |
| GET    | `/api/v1/summary` | Aggregate stats: `active_sessions`, `state_breakdown`, `total_agents`, `limits`. |
| GET    | `/api/v1/limits` | Latest cached usage-limits snapshot. 404 when `fetch_limits` is disabled or unavailable. |
| GET    | `/api/v1/limits/history` | Historical limits snapshots. `?since=T&limit=N&offset=N`. |

### Platform-raw (Claude)

| Method | Path | Purpose |
|---|---|---|
| GET    | `/api/v1/claude/sessions` | Raw `claude_sessions` rows. |
| GET    | `/api/v1/claude/events`   | Raw `claude_events` rows. `?session_id=X&event_name=X&since=T&until=T&limit=N&offset=N`. |

### Ingestion (internal)

| Method | Path | Purpose |
|---|---|---|
| POST   | `/hooks/claude`     | Hook relay POSTs every Claude Code hook event here. Body: hook JSON enriched with `pid` and `source_system`. |
| POST   | `/statusline/claude`| Statusline relay POSTs cost/token/context snapshots. Stores the row even when the session is unknown; returns `{"status":"deferred"}` in that case. |
| POST   | `/costs/claude`     | Stub. Returns 501. Reserved for future external cost feeds. |

## Response shapes

### `SessionResponse` (`GET /api/v1/sessions` element)

```json
{
  "session_id": "a415cfed-95a4-...",
  "platform": "claude",
  "pid": 68718,
  "cwd": "/path/to/project",
  "entrypoint": "vscode",
  "started_at": 1776703873232,
  "last_event": "PostToolUse",
  "last_tool": "Read",
  "last_event_at": 1776947339.99,
  "pid_alive": true,
  "ended_at": null,
  "branch": "",
  "source_system": "host.example",
  "cost_usd": 83.58,
  "prior_cost_usd": 82.54,
  "context_used_pct": 6,
  "model_name": "Opus 4.6 (1M context)",
  "total_input_tokens": 292478,
  "total_output_tokens": 512427,
  "lines_added": 6269,
  "lines_removed": 944,
  "agent_count": 0,
  "derived_state": "working"
}
```

`cost_usd` is the **process-cumulative** cost reported by Claude Code's
statusline (see runtime reference) — its value can repeat across rows
that share the same `pid` and `source_system`. `prior_cost_usd` is the
max statusline `cost_usd` recorded **before local midnight** for this
session, used to derive "today's cost" as
`cost_usd - prior_cost_usd`. `started_at` is milliseconds (the only
ms field in this surface).

### `SessionDetailResponse` (`GET /api/v1/sessions/{session_id}`)

Same fields as `SessionResponse` plus:

```json
{
  "agents": [
    {
      "agent_id": "agent-uuid",
      "session_id": "a415cfed-...",
      "platform": "claude",
      "agent_type": "Explore",
      "last_event": "SubagentStart",
      "last_tool": null,
      "last_event_at": 1776947339.99,
      "started_at": 1776947320.10,
      "ended_at": null,
      "derived_state": "working"
    }
  ]
}
```

`prior_cost_usd` is computed from this session's statusline history
(not the broader process group's). If two sessions share a process,
each computes `prior_cost_usd` independently from its own rows; in
practice the values match because the statusline cost is
process-cumulative.

### `EventResponse` (from `/sessions/{id}/events`)

```json
{
  "id": 4123,
  "session_id": "a415cfed-...",
  "platform": "claude",
  "agent_id": null,
  "event_name": "PreToolUse",
  "tool_name": "Bash",
  "cwd": "/path/to/project",
  "received_at": 1776947339.99
}
```

### `SummaryResponse`

```json
{
  "active_sessions": 4,
  "state_breakdown": {"working": 2, "idle": 2},
  "total_agents": 5,
  "limits": { "...": "see GET /api/v1/limits below" }
}
```

`active_sessions` counts sessions with `pid_alive=true` and
`ended_at` null, **not processes**. Two sessions belonging to the same
process count as two.

### `GET /api/v1/limits`

```json
{
  "available": true,
  "fetched_at": 1776052424.367,
  "age_seconds": 0.0,
  "stale": false,
  "refresh_after": 1776052544.367,
  "five_hour": {"utilization": 11.0, "resets_at": "2026-04-13T07:00:00+00:00"},
  "seven_day": {"utilization": 8.0, "resets_at": "2026-04-19T13:00:00+00:00"}
}
```

`GET /api/v1/limits/history` returns a list of
`{id, fetched_at, age_seconds, data}` where `data` is the parsed
`raw_response` JSON.

### `POST /api/v1/sessions/{session_id}/clear`

```json
{ "status": "cleared", "session_id": "a415cfed-..." }
```

Side effects: ends every active agent for the session, inserts a
synthetic `clear_state` row in `claude_events`, broadcasts
`session_cleared` over WebSocket. 404 when the session is unknown.

### Ingestion responses

`POST /hooks/claude` → `{"status": "ok"}` (or
`{"status":"error","reason":"database error"}`).

`POST /statusline/claude` →
- `{"status": "ok"}` when the session exists
- `{"status": "deferred", "reason": "session not found"}` when the row
  is stored but no matching session row exists yet
- `{"status": "ignored", "reason": "no session_id"}` when the payload
  has no `session_id`

`POST /costs/claude` → HTTP 501.

## Derived state

The REST API includes a `derived_state` field on each session.
Computed from `last_event` and `last_tool` at query time. **This v1
mapping diverges from the authoritative
[state-transitions reference](../reference/state-transitions.md)
— see the wart list below.**

| event_name | tool_name | derived_state |
|---|---|---|
| `PreToolUse`        | `AskUserQuestion`  | `awaiting_input` |
| `PreToolUse`        | (any other)        | `working` |
| `PermissionRequest` | `AskUserQuestion`  | `awaiting_input` |
| `PermissionRequest` | (any other)        | `permission_required` |
| `UserPromptSubmit`  | —                  | `working` |
| `PostToolUse`       | —                  | `working` |
| `Stop`              | —                  | `idle` |

When a session has active agents, the effective `derived_state` is
the highest-priority state across the main session and all agents.
Priority (highest first):
`permission_required > awaiting_input > working > ready > idle`.

## Pagination

`/sessions/{id}/events`, `/limits/history`, and `/claude/events`
accept `limit`, `offset`, and `since`. Default `limit=100`. `since` is
Unix seconds (float), inclusive lower bound on the row's primary
timestamp. There is no token-based pagination.

## Error model

FastAPI default — `{"detail": "..."}` body with appropriate status
codes. Intentional 4xx in this surface: `404 Not Found` when a
`session_id` is unknown. Database errors during ingestion log
server-side and return a `{"status": "error"}` body with 200 OK
(non-retrying convention with the relay scripts).

## Known issues / v1 warts

- **Mixed time units.** `started_at` is milliseconds; everything else
  is seconds.
- **Sessions are flat.** A user running multiple sessions inside one
  Claude process (via `/clear`) sees them as independent rows.
  Cumulative metrics like `cost_usd` are duplicated across those rows
  because Claude Code reports them per-process. v2 fixes this by
  introducing a process entity.
- **`prior_cost_usd` is per-session, not per-process.** Across a
  `/clear` boundary the new session has its own statusline rows
  starting fresh; `prior_cost_usd` for that session reads the
  pre-midnight row from this session_id only. Combined with the
  process-cumulative nature of `cost_usd`, the math `cost_usd -
  prior_cost_usd` is **only correct if the session lasted the full
  day** — sessions started today after a previous session in the
  same process get inflated "today's cost" estimates because their
  baseline is too low.
- **No process address.** No way to say "give me everything for
  process X" without grouping the session list yourself.
- **Statusline endpoint silently drops the unknown-session case.**
  Returns 200 with `{"status": "deferred"}` and no broadcast — a
  client streaming WebSocket has no signal that the row arrived.
- **Resume into a new PID overwrites session state.** `claude
  --resume` from a new shell overwrites `pid` (and possibly other
  process-level fields) on the existing session row; the original
  process lifetime is lost from current-state queries (still
  recoverable from `claude_events.raw_payload`).
- **`derive_state` diverges from the dashboard spec.** v1 maps
  `Stop → idle` and orders priority `working > ready`. The
  authoritative claude-dashboard state machine has `Stop → ready`
  (transient — only goes to `idle` when the user clicks the row)
  and `ready > working` (the main session being done outranks a
  background agent still working). The v1 mapping was hardcoded
  before the spec stabilized; v2 corrects it.
