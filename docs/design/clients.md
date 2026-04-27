# Client Guide

How to write a client against AgentPulse. Focused, hands-on. For
the full surface, see [`api.md`](api.md), [`websocket.md`](websocket.md),
and [`../reference/state-transitions.md`](../reference/state-transitions.md).
For the system-level view of how data flows end-to-end, see
[`ux.md`](ux.md).

## Connecting

| Surface | URL |
|---|---|
| REST | `http://{host}:{port}/api/v2/...` |
| WebSocket | `ws://{host}:{port}/ws/v2` |

Default `127.0.0.1:17385` (configurable in
`~/.claude/agentpulse/config.json`).

WebSocket: receive-only, JSON text frames, no server-initiated
pings. Disable client-side ping expectations or send your own
keep-alives.

## Conceptual model

- A **process** is one Claude Code OS instance, identified by
  `(pid, source_system, cwd)`. Cumulative cost / tokens / lines
  live on the process.
- A process hosts one or more **sessions**. A `/clear` mints a new
  session within the same process. Per-session fields (last event,
  context %, model) reset.
- A session can span multiple **epochs** if it's resumed into a new
  pid (`claude --resume`). Each `(session_id, pid)` span is one
  epoch.
- An **agent** is a subagent within a session.

Server-derived ids appear on every response and frame:
`process_id`, `epoch_id`. Treat as opaque.

For the upstream Claude Code semantics behind these, see
[`../reference/claude-runtime.md`](../reference/claude-runtime.md).

## Bootstrap

```
1. Open WebSocket /ws/v2. Buffer frames; don't process yet.
2. GET /api/v2/processes?active=true   → live processes (sessions nested)
3. GET /api/v2/limits                  → optional, if you display them
4. Apply buffered frames + subsequent live frames.
```

That's it. The processes endpoint includes nested sessions, agents,
and `cost_by_day` — you don't need a second call.

## Reconnect

WebSocket connections drop. **Reconnect by re-running bootstrap.**
The server provides no replay; a fresh `GET /api/v2/processes` is a
complete snapshot, which is all a live dashboard needs.

`log_id` on every frame is still useful for clients that page raw
history (`GET /api/v2/log/{table}`) and want to dedupe — it is not
required for live reconnect.

## When to call REST after bootstrap

Only at three boundary moments. The WebSocket carries everything
else:

1. **New session discovered.** A `hook_logged` arrives for a
   `session_id` you haven't seen. Fetch
   `GET /api/v2/processes/{process_id}` once to pick up
   `cost_by_day` for the enclosing process. Cache locally.
2. **Day boundary.** At local midnight, refresh `cost_by_day` for
   every active process. Today's bucket rolls; yesterday's becomes
   prior-day. Apply subsequent `statusline_logged` deltas to the
   new day's bucket yourself.
3. **Suspected drift (optional).** On a long cadence (e.g. 5 min),
   `GET /api/v2/processes?active=true` and compare each process's
   `last_activity_at` against the last frame you saw for that
   `process_id`. A drift larger than one statusline cadence
   suggests missed messages — re-bootstrap. Guardrail, not a
   primary loop.

Don't poll on a tight cadence. The wire is authoritative between
boundary moments.

## WebSocket frames — what to do

Four message types. Common envelope: `type`, `platform`,
`timestamp`, `log_id`. Session-scoped frames also carry
`process_id`, `epoch_id`, `pid`, `source_system`, `cwd`,
`session_id`. Process-scoped frames carry process keys without
`session_id`. See [`websocket.md`](websocket.md) for the full
field list.

### `hook_logged`

The catch-all for hook events. Dispatch on `event_name`.

| `event_name` | What the client typically does |
|---|---|
| `SessionStart`      | Add the session under its process. |
| `SessionEnd`        | Mark the session terminal; keep history. |
| `UserPromptSubmit`  | Set state to `working`; clear stale agents from prior turn. |
| `PreToolUse`        | Update state per the derived-state mapping; show `tool_name`. |
| `PostToolUse`       | Set state to `working`. |
| `PermissionRequest` | Set state to `permission_required` (or `awaiting_input` for `AskUserQuestion`). |
| `Stop`              | Set state to `ready`. |
| `Notification`      | Surface the message somewhere. |
| `SubagentStart`     | Add the agent to the session's agent list. |
| `SubagentStop`      | Mark the agent ended. |
| `clear_state`       | Reset session display to idle. Synthetic, written by `POST .../clear`. |

Ignore unknown `event_name` values — Claude Code adds events.

### `statusline_logged`

A cost / token / context snapshot. Replace (don't accumulate) the
process's `cost_usd`, `total_input_tokens`, `total_output_tokens`,
`lines_added`, `lines_removed`. Replace the session's
`context_used_pct` and `model_name`.

`cost_usd` is **process-cumulative**, not session-cumulative — it
resets to $0 in a new pid after `claude --resume`.

`session_total_cost_usd` is the **session-cumulative** total derived
server-side (sum across all instance windows in the session,
including this row). Replace the session's running total with this
value — no client-side aggregation, no REST refetch on resume. Same
field as `SessionResponse.total_cost_usd` from REST.

`session_today_cost_usd` is **today's bucket** (server local time)
of the session cost. Same value the client would derive by plucking
today's key from `cost_by_day` on REST. Drives the "spend so far
today" indicator without forcing the client to track yesterday's
total or refetch at midnight.

`cost_by_day` is **not on the wire** — sending the full map on every
statusline would grow unbounded for long-running sessions. If you
need historical breakdown, refetch on a coarse cadence:

```
GET /api/v2/sessions/{session_id}     # has cost_by_day
GET /api/v2/processes/{process_id}    # has cost_by_day for the process
```

A reasonable cadence is once per day shortly after midnight — that's
when yesterday's bucket finalizes and a new today bucket appears.

A `statusline_logged` may arrive for a session_id with no prior
`hook_logged` (statusline can race ahead of hooks). Create a
pending session entry on first sight; the eventual hook fills in
the rest.

### `pid_death_logged`

Process-scoped. One frame, but the implications fan out:

- Mark every session under `process_id` as ended.
- Mark every active agent under those sessions as ended.
- Set the process `pid_alive = false`.
- `ended_at` for any open epoch in the process = `observed_at`.

This replaces N session-ended messages from older designs — clients
fan out themselves now.

### `api_limits_logged`

Global (no process_id). Update the displayed limits. Use
`five_hour_resets_at` and `seven_day_resets_at` (epoch seconds) for
"resets in X hours" UI.

## REST — what you actually call

Most clients use four endpoints in steady state:

| When | Endpoint |
|---|---|
| Bootstrap / reconnect | `GET /api/v2/processes?active=true` |
| New session discovered | `GET /api/v2/processes/{process_id}` (one-shot, for `cost_by_day`) |
| Day boundary (midnight) | `GET /api/v2/processes/{process_id}` per active process (refresh `cost_by_day`) |
| Optional aggregate | `GET /api/v2/summary` |

Less commonly:

| When | Endpoint |
|---|---|
| Drill into one session | `GET /api/v2/sessions/{session_id}` (epochs + agents nested) |
| Show event history | `GET /api/v2/sessions/{session_id}/events?limit=N` |
| Show statusline history | `GET /api/v2/sessions/{session_id}/statuslines?limit=N` |
| Show usage limits | `GET /api/v2/limits` |
| Reset a stuck session | `POST /api/v2/sessions/{session_id}/clear` |

`POST /sessions/{id}/clear` is the only mutation you'll call. Use
it when the user signals "Claude was interrupted, but the dashboard
still shows working" — see the runtime reference for why
interrupts produce no hook.

For the full endpoint list, query parameters, and response shapes,
see [`api.md`](api.md).

## Derived state

| event_name          | tool_name         | session state         |
|---------------------|-------------------|-----------------------|
| `PreToolUse`        | `AskUserQuestion` | `awaiting_input`      |
| `PreToolUse`        | (any other)       | `working`             |
| `PermissionRequest` | `AskUserQuestion` | `awaiting_input`      |
| `PermissionRequest` | (any other)       | `permission_required` |
| `UserPromptSubmit`  | —                 | `working`             |
| `PostToolUse`       | —                 | `working`             |
| `Stop`              | —                 | `ready`               |
| `SessionEnd`        | —                 | `null` (terminal)     |
| `clear_state`       | —                 | `idle`                |
| (no events yet)     | —                 | `null` (unknown)      |

Agents are simpler: only `working`, `permission_required`, or
`awaiting_input` (they never see `Stop`).

Effective state for the process display = highest priority across
the current session and any active agents.
Priority (highest first):
`permission_required > awaiting_input > ready > working > idle`.

`ready` outranks `working` deliberately — the main session having
finished outranks a background agent still ticking. The user
should see "ready."

`idle` is reached only by `clear_state`. After `Stop`, the server
returns `ready`; clients transition `ready → idle` on user
acknowledgment (e.g. clicking the row).

Source of truth:
[`../reference/state-transitions.md`](../reference/state-transitions.md).

## Common patterns

### Filtering by process

Every session-scoped frame carries `process_id`. To track one
process: filter on `process_id`. Don't reconstruct it from
`(pid, source_system, cwd)` — the server's id is authoritative.

### Filtering by source system

Multi-machine deployments use `source_system` (hostname). The
field is on every session/process-scoped frame and on every
process / session response. Filter client-side.

### Sparse data on first sight

A session can show up first via `statusline_logged` (no hook yet).
Display it tentatively; populate the rest on the first
`hook_logged`.

### Limits

`api_limits_logged` only fires on successful fetches. If you care
about staleness, watch the gap between consecutive
`received_at` values; long gaps may mean fetcher failures (the
server logs them, doesn't broadcast). The
[REST `/api/v2/limits`](api.md) response carries `stale` and
`age_seconds` for explicit checking.

## Don't do

- **Don't accumulate `cost_usd`** across `statusline_logged`
  frames. It's cumulative; replace.
- **Don't sum `cost_usd` across processes** to get a session total.
  Use `session_total_cost_usd` from the same frame — the server
  already did the instance-windowed aggregation correctly.
- **Don't compute `process_id` yourself.** It's an opaque
  server-derived hash. Echo what the server gives you.
- **Don't rely on `event_name` for cleanup.** When
  `pid_death_logged` arrives, the process is gone — even if no
  `SessionEnd` ever fired.
- **Don't expect server pings.** None. Send client-side keep-alives
  if your stack needs them.
- **Don't enum-validate `event_name` / `tool_name` / `entrypoint`.**
  Claude Code adds values; tolerate unknowns.
- **Don't request raw payloads on the live path.** Wire frames
  exclude `raw_payload` for size. Use
  `?include=raw` on REST endpoints when you need it.
