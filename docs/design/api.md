# REST API

REST surface that sits on top of the data model in
[`database.md`](database.md). Companion to
[`websocket.md`](websocket.md). Describes the shape only — no
implementation steps.

## Principles

1. **Match the data model.** The API exposes derived concepts
   (process, session, agent, epoch) without pretending they are
   stored. Reads are projections over `claude_log_*` tables.
2. **One time unit everywhere.** Every timestamp is Unix epoch
   seconds (float). No milliseconds anywhere on the wire.
3. **Process and session are addressable as separate resources.**
   `/processes` and `/sessions` both exist; nesting works in detail
   views.
4. **Process epochs are first-class.** A session that resumed into a
   new PID is a session spanning two epochs. The API surfaces that
   history.
5. **Logs are scannable.** Each log table has a corresponding raw
   read endpoint with consistent pagination semantics. Clients that
   need the raw record never have to parse a derived view.
6. **Stable, opaque ids for derived entities.** `process_id` and
   `epoch_id` are server-computed, stable for the lifetime of the
   underlying logs, opaque to clients. The composite natural key
   stays available on every response.

## Versioning

All paths under `/api/v2/`. Ingestion endpoints (`/hooks/claude`,
`/statusline/claude`) are unversioned internal contracts. A future
`/v3` may add sibling ingestion endpoints; the existing ones stay
in place.

## Time units

**Every** timestamp is `REAL`-valued **Unix epoch seconds**.
Resolution is fractional seconds, matching the database. No fields
expressed in milliseconds. Clients render however they like.

The only string timestamps are upstream-OAuth `resets_at` values
inside the `limits` payload, which arrive as ISO-8601 from the
Anthropic API and are echoed through unchanged. Everything we
control is seconds.

## Conceptual model

Upstream Claude Code runtime semantics — process, session, `/clear`,
resume — live in [`../reference/claude-runtime.md`](../reference/claude-runtime.md).
The data model in [`database.md`](database.md) builds on those plus
the **epoch** concept this API surfaces:

- **Process.** Identified by `(pid, source_system, cwd)` plus a
  time-window discriminator that separates PID-reused instances.
- **Session.** Identified by `session_id`. May span multiple
  process epochs (resume into a new pid).
- **Process epoch.** A contiguous span of rows sharing the same
  `(session_id, pid)`. A pid change within a session_id starts a new
  epoch — the hallmark of `claude --resume` into a new process.
- **Agent.** Identified by `agent_id`. Lives within one session.

The natural identity composite for a hook event is
`(pid, session_id, source_system, cwd, agent_id)`. Statusline events
have no `agent_id` dimension (only the main CLI renders).

## Entity ids

Two server-computed ids appear on responses. Both are opaque to
clients — derive once, echo back for routing.

| Id | Derived from | Stable across | Notes |
|---|---|---|---|
| `process_id` | `(source_system, pid, cwd, earliest_received_at)` floored to seconds | The lifetime of the underlying logs | PID reuse produces a different id because `earliest_received_at` differs. |
| `epoch_id` | `(session_id, pid, source_system, cwd, earliest_received_at)` | The lifetime of the underlying logs | One epoch per `(session_id, pid)` span. |

Algorithm: `sha256(f"{a}|{b}|{c}|{d}".encode()).hexdigest()[:16]`.
Documented for verification; clients should treat the strings as
opaque.

## Endpoint catalog

### Resources

| Method | Path | Purpose |
|---|---|---|
| GET    | `/api/v2/processes` | List processes. Filters: `?active=true`, `?source_system=X`, `?cwd=X`, `?since=T`. |
| GET    | `/api/v2/processes/{process_id}` | Process detail with all sessions and epochs nested. |
| GET    | `/api/v2/processes/{process_id}/events` | Hook events for the whole process (every session in the group). Pagination: `?since=T&until=T&limit=N&offset=N`. |
| GET    | `/api/v2/processes/{process_id}/statuslines` | Statusline log for the process. Pagination as above. |
| GET    | `/api/v2/sessions` | List sessions. Filters: `?active=true`, `?process_id=X`, `?since=T`. |
| GET    | `/api/v2/sessions/{session_id}` | Session detail with epochs and agents nested. |
| GET    | `/api/v2/sessions/{session_id}/epochs` | Process epochs for this session. |
| GET    | `/api/v2/sessions/{session_id}/events` | Hook events for this session. Pagination as above. Filters: `?event_name=X`, `?tool_name=X`, `?agent_id=X`. |
| GET    | `/api/v2/sessions/{session_id}/statuslines` | Statusline log for this session. |
| GET    | `/api/v2/sessions/{session_id}/agents` | Agents for this session. |
| GET    | `/api/v2/agents/{agent_id}` | Agent detail. |
| GET    | `/api/v2/agents/{agent_id}/events` | Hook events scoped to one agent. |
| POST   | `/api/v2/sessions/{session_id}/clear` | Insert a synthetic `clear_state` row in `claude_log_hooks` and broadcast `session_cleared`. Used when Claude is interrupted with no events firing. |
| GET    | `/api/v2/limits` | Latest cached usage-limits snapshot. 404 when disabled or unavailable. |
| GET    | `/api/v2/limits/history` | Historical limits snapshots. Pagination as above. |
| GET    | `/api/v2/summary` | Aggregate stats across the running deployment. |

### Raw log streams

One read endpoint per log table. Append-only, naturally ordered by
`id`. Useful for clients that want the underlying row without any
derivation.

| Method | Path | Source table | Pagination |
|---|---|---|---|
| GET | `/api/v2/log/hooks` | `claude_log_hooks` | `?since=T&until=T&session_id=X&pid=N&source_system=X&cwd=X&agent_id=X&event_name=X&tool_name=X&limit=N&offset=N` |
| GET | `/api/v2/log/statuslines` | `claude_log_statuslines` | `?since=T&until=T&session_id=X&pid=N&source_system=X&cwd=X&limit=N&offset=N` |
| GET | `/api/v2/log/pid-deaths` | `claude_log_pid_deaths` | `?since=T&until=T&pid=N&source_system=X&cwd=X&limit=N&offset=N` |
| GET | `/api/v2/log/api-limits` | `claude_log_api_limits` | `?since=T&until=T&limit=N&offset=N` |

`since` / `until` are inclusive bounds on the row's primary
timestamp — `received_at` for wire-sourced logs, `observed_at` for
`pid-deaths`.

### Ingestion (internal, unversioned)

The hook and statusline relays write into `claude_log_hooks` and
`claude_log_statuslines` respectively.

| Method | Path | Notes |
|---|---|---|
| POST | `/hooks/claude` | Append-only insert into `claude_log_hooks`. Always 200 on success. |
| POST | `/statusline/claude` | Append-only insert into `claude_log_statuslines`. Returns 200 with `{"status":"ok"}` even when the session_id has no matching hook yet — the row is data, not a request to mutate state. |

Ingestion is purely append-only — every write either succeeds or
returns an error. No "deferred" status.

## Response shapes

### `ProcessResponse`

```json
{
  "process_id": "a1b2c3d4e5f60718",
  "pid": 68718,
  "source_system": "host.example",
  "cwd": "/path/to/project",
  "platform": "claude",
  "entrypoint": "vscode",
  "claude_version": "2.1.117",
  "permission_mode": "acceptEdits",
  "started_at": 1776703873.232,
  "last_activity_at": 1776947339.99,
  "ended_at": null,
  "pid_alive": true,
  "current_session_id": "a415cfed-95a4-...",
  "derived_state": "working",
  "cost_usd": 83.58,
  "cost_by_day": {
    "2026-04-22": 82.54,
    "2026-04-23": 1.04
  },
  "total_input_tokens": 292478,
  "total_output_tokens": 512427,
  "lines_added": 6269,
  "lines_removed": 944,
  "model_id": "claude-opus-4-7[1m]",
  "model_name": "Opus 4.7 (1M context)",
  "session_count": 2,
  "epoch_count": 2,
  "sessions": [ /* SessionSummary[], present on /processes/{id} only */ ]
}
```

Notes:
- `started_at` = `MIN(received_at)` across the process's hook log
  rows. `last_activity_at` = `MAX(received_at)` across hooks and
  statuslines. `ended_at` = matching `claude_log_pid_deaths.observed_at`
  if any, else null (alive or last-known-active).
- `pid_alive`: `true` when there is no death record for this process.
  False once the watcher writes a death row.
- `derived_state`: computed from the latest hook event for the
  current session and any active agents. Mapping table below.
- `cost_*`, tokens, and lines: computed from
  `claude_log_statuslines` rows scoped to the process's
  `(pid, source_system, cwd)` and time window. `cost_by_day` keys are
  `YYYY-MM-DD` in server local time.
- `model_id` / `model_name`: from the latest statusline row for the
  process.
- `session_count` / `epoch_count`: convenience counts. Useful for
  list views without paying the cost of fetching the full nested
  arrays.

`GET /api/v2/processes` returns a list of `ProcessResponse` **without**
the `sessions` array. `GET /api/v2/processes/{process_id}` includes
the array.

### `SessionResponse`

```json
{
  "session_id": "a415cfed-95a4-...",
  "platform": "claude",
  "process_id": "a1b2c3d4e5f60718",
  "pid": 68718,
  "source_system": "host.example",
  "cwd": "/path/to/project",
  "started_at": 1776946322.574,
  "last_activity_at": 1776947339.99,
  "ended_at": null,
  "last_event": "PostToolUse",
  "last_tool": "Read",
  "last_event_at": 1776947339.99,
  "context_used_pct": 6,
  "context_window_size": 1000000,
  "model_id": "claude-opus-4-7[1m]",
  "model_name": "Opus 4.7 (1M context)",
  "agent_count": 0,
  "derived_state": "working",
  "epoch_count": 1,
  "epochs": [ /* EpochResponse[], present on /sessions/{id} only */ ],
  "agents": [ /* AgentResponse[], present on /sessions/{id} only */ ]
}
```

Notes:
- `process_id` is the **current** epoch's process. A session that
  spanned two processes (resume into new pid) reports the most recent
  one here; the full history is in `epochs`.
- `pid` / `source_system` / `cwd` mirror the current epoch — clients
  that already know the parent process can ignore these and use
  `process_id`.
- `last_event` / `last_tool` / `last_event_at`: from the latest hook
  event for this session.
- `started_at` and `last_activity_at` are seconds; v1's millisecond
  `started_at` is gone.

### `EpochResponse`

```json
{
  "epoch_id": "9f8e7d6c5b4a3210",
  "session_id": "a415cfed-95a4-...",
  "process_id": "a1b2c3d4e5f60718",
  "pid": 68718,
  "source_system": "host.example",
  "cwd": "/path/to/project",
  "entrypoint": "vscode",
  "started_at": 1776946322.574,
  "ended_at": null,
  "event_count": 142
}
```

One row per `(session_id, pid)` span. `ended_at` is null while the
epoch is open; otherwise it is `MAX(received_at)` for the span,
firmed up by a matching `claude_log_pid_deaths.observed_at` when one
exists.

### `AgentResponse`

```json
{
  "agent_id": "agent-uuid",
  "session_id": "a415cfed-95a4-...",
  "process_id": "a1b2c3d4e5f60718",
  "platform": "claude",
  "agent_type": "Explore",
  "started_at": 1776947320.10,
  "ended_at": null,
  "last_event": "SubagentStart",
  "last_tool": null,
  "last_event_at": 1776947320.10,
  "derived_state": "working"
}
```

`agent_type` comes from the introducing `SubagentStart` row;
subsequent rows for the same `agent_id` may leave it null, but the
response always reports the type that was originally captured.

### `EventResponse` (`claude_log_hooks` projection)

```json
{
  "id": 4123,
  "session_id": "a415cfed-...",
  "process_id": "a1b2c3d4e5f60718",
  "epoch_id": "9f8e7d6c5b4a3210",
  "agent_id": null,
  "platform": "claude",
  "pid": 68718,
  "source_system": "host.example",
  "cwd": "/path/to/project",
  "event_name": "PreToolUse",
  "tool_name": "Bash",
  "permission_mode": "acceptEdits",
  "claude_version": "2.1.117",
  "received_at": 1776947339.99,
  "received_by": "hook-endpoint"
}
```

`raw_payload` is **omitted by default** to keep responses small. Pass
`?include=raw` on any events endpoint to receive the full hook JSON
inlined as a **JSON string** in a `raw_payload` field — clients
that need it call `JSON.parse()` themselves. Inlining as a string
avoids any double-encoding concerns and matches what's stored in
the database column.

The raw read endpoint `/api/v2/log/hooks` includes `raw_payload`
by default (also as a string). All other endpoints default to
omitting it.

### `StatuslineResponse` (`claude_log_statuslines` projection)

```json
{
  "id": 891,
  "session_id": "a415cfed-...",
  "process_id": "a1b2c3d4e5f60718",
  "epoch_id": "9f8e7d6c5b4a3210",
  "platform": "claude",
  "pid": 68718,
  "source_system": "host.example",
  "cwd": "/path/to/project",
  "project_dir": "/path/to/project",
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
  "received_by": "statusline-endpoint"
}
```

Mirrors `claude_log_statuslines` columns one-to-one (plus derived
`process_id` / `epoch_id`). All time fields are seconds.
`raw_payload` available with `?include=raw`.

### `PidDeathResponse` (`claude_log_pid_deaths`)

```json
{
  "id": 14,
  "pid": 68718,
  "source_system": "host.example",
  "cwd": "/path/to/project",
  "process_id": "a1b2c3d4e5f60718",
  "observed_at": 1776948000.42,
  "observed_by": "pid-watcher"
}
```

### `LimitsResponse` (`/limits`)

```json
{
  "available": true,
  "fetched_at": 1776052424.367,
  "age_seconds": 0.0,
  "stale": false,
  "refresh_after": 1776052544.367,
  "five_hour": {
    "utilization": 11.0,
    "resets_at_seconds": 1776920400.0,
    "resets_at": "2026-04-13T07:00:00+00:00"
  },
  "seven_day": {
    "utilization": 8.0,
    "resets_at_seconds": 1776981600.0,
    "resets_at": "2026-04-19T13:00:00+00:00"
  }
}
```

Both `resets_at_seconds` (epoch seconds, our standard unit) and the
upstream ISO string `resets_at` (passthrough from the OAuth API) are
returned. Clients should prefer `resets_at_seconds`.

### `SummaryResponse`

```json
{
  "active_processes": 3,
  "active_sessions": 4,
  "active_agents": 5,
  "active_epochs": 4,
  "state_breakdown": {"working": 2, "idle": 1},
  "limits": { /* LimitsResponse, or null */ }
}
```

`active_processes` = processes with no `claude_log_pid_deaths`
row. PID alive ⇒ process active. Whether individual sessions in
the group have ended is irrelevant.

`active_sessions` = sessions whose latest hook is not `SessionEnd`
AND whose enclosing process is active. A session with zero hooks
(statusline arrived, no hook yet) **is** active — it has a valid
`session_id`, the process is alive, and there is no `SessionEnd`
to mark it ended.

`active_agents` = agents whose latest hook is not `SubagentStop`
AND whose enclosing session is active.

`active_epochs` = epochs whose `(session_id, pid)` span has no
`claude_log_pid_deaths` row matching the epoch's
`(pid, source_system, cwd)`.

### `POST /api/v2/sessions/{session_id}/clear`

```json
{ "status": "cleared", "session_id": "a415cfed-...", "received_at": 1776947400.0 }
```

Side effect: appends one synthetic row to `claude_log_hooks` with
`event_name = "clear_state"`.

**The synthetic row MUST carry the same identity as a real hook
row.** The writer looks up the most recent prior `claude_log_hooks`
row for `session_id` and copies forward `pid`, `source_system`,
`cwd`, and `entrypoint`. `agent_id` is null. `raw_payload` is a
short JSON object with the keys the writer derived plus
`{"_synthetic": "clear-endpoint"}` — not the empty `{}` v1 used,
which produced pid-less rows that orphaned in process derivation.

The synthetic clear is rejected (404) when no prior hook exists for
the session — there's nothing to copy keys from, and clearing a
session the server hasn't observed is meaningless.

No agent rows are mutated — agents that were active at the time
will look "stuck" in the derived view until a later signal closes
them. (Append-only invariant: the design deliberately does not
write `SubagentStop` events the user did not emit. Consumers that
need a stronger reset can ignore active agents older than the most
recent `clear_state` for the session.)

The clear remains a 404 if the session is unknown.

### Errors

FastAPI default body: `{"detail": "..."}`. Status codes used:

- `200` on every successful read/write
- `400` on bad query parameters (e.g. malformed `since`)
- `404` when a `process_id`, `session_id`, `agent_id`, or `epoch_id`
  is unknown
- `503` when `fetch_limits` is disabled or the OAuth fetch is
  failing — no longer a 404 (limits not being available is an
  operational state, not a missing resource)

## Filtering and pagination

All list / log endpoints share one pagination shape:

| Param | Type | Notes |
|---|---|---|
| `since` | float seconds | Inclusive lower bound on the row's primary timestamp. |
| `until` | float seconds | Inclusive upper bound. |
| `limit` | int (default 100, max 1000) | Page size. |
| `offset` | int (default 0) | Skip count. |

Domain filters (e.g. `event_name`, `tool_name`, `agent_id`) are
documented per endpoint above. Filters compose; missing filters mean
"any value." Time filters apply to `received_at` for all wire-sourced
logs and to `observed_at` for `pid-deaths`.

There is no token-based pagination. Clients that want strict
"continue from where I stopped" semantics should pass the previous
page's max `id` via `since=<that_row.received_at>` and discard the
boundary row themselves.

## Derived state

The authoritative session/agent state machine for Claude Code is
documented in
[`../reference/state-transitions.md`](../reference/state-transitions.md)
— that doc is the source of truth for what each state means and how
transitions work. Summarized here for self-containment.

### Session (main process) mapping

| event_name          | tool_name         | derived_state         |
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

### Agent mapping

Agents have a simpler lifecycle — they never see `Stop`, so they
never enter `ready` or `idle`. Per
[the state-transitions reference](../reference/state-transitions.md),
agents are only ever `working`, `permission_required`, or
`awaiting_input`.

| event_name          | tool_name         | agent_derived_state   |
|---------------------|-------------------|-----------------------|
| `SubagentStart`     | —                 | `working`             |
| `PreToolUse`        | `AskUserQuestion` | `awaiting_input`      |
| `PreToolUse`        | (any other)       | `working`             |
| `PermissionRequest` | `AskUserQuestion` | `awaiting_input`      |
| `PermissionRequest` | (any other)       | `permission_required` |
| `PostToolUse`       | —                 | `working`             |
| `SubagentStop`      | —                 | `null` (terminal)     |

### Effective state

Process effective state = highest-priority across the current
session's state and any active-agent states.

Priority (highest first):
`permission_required > awaiting_input > ready > working > idle`.

`ready` outranks `working` deliberately: when the main session has
finished but a background agent is still ticking, the user should
see "ready" — the main thing they care about is done. The dashboard
spec is explicit on this and AgentPulse should match.

### Notes on individual states

- **`ready`** is what `Stop` produces. Semantically: "main process
  finished a turn; nothing is blocking; the next move is yours."
  `ready` is transient — clients typically transition to `idle` once
  the user has acknowledged it (e.g. clicked the row). AgentPulse
  itself never returns `idle` from a hook event — `idle` only comes
  from the synthetic `clear_state` row written by
  `POST /api/v2/sessions/{id}/clear`.
- **`null`** appears in two distinct cases: terminal (after
  `SessionEnd` or `SubagentStop`) and unknown (no hook events yet).
  The session's `last_event` field disambiguates: null
  `last_event` → unknown; non-null with terminal event → terminal.
- **Interrupts produce no transition.** When the user presses ESC
  mid-turn, no hook fires (see `../reference/claude-runtime.md`),
  so the derived state stays at whatever the last hook said —
  typically `working`. Recovery is via `POST .../clear`. This is
  upstream behavior, not an AgentPulse choice.

## Id stability

Process and epoch ids are derived from log content. A reader that
recomputes them from the same logs gets the same ids. There is no
write-side state to keep them stable; they are deterministic
projections.

Adding a new field to the derivation (e.g. account scoping when
multi-account lands — see the Single Anthropic account constraint
in `database.md`) changes every id. Such changes are breaking and
warrant `/api/v3/`.
