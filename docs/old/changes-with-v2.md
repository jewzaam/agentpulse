# Changes with v2

Meta-level v1→v2 deltas. What goes away, what shows up, what
changes shape. The design docs in `docs/design/` describe v2 as it
should be — they don't carry transition cruft. This page is where
the transition story lives.

When v2 ships and v1 is removed, this whole `docs/old/` tree
(including this file) goes with it. The design docs stay clean.

## Removed

- **`POST /costs/claude` endpoint and the `claude_costs` table.**
  v1 had both as a stub for a future external cost feed. v2 only
  describes `claude_log_external_costs` as a *future* table to land
  alongside a real writer; no endpoint until that writer exists.
- **`prior_cost_usd` field.** Replaced by process-level
  `cost_by_day` on the REST process body. v1's `prior_cost_usd`
  was per-session and didn't compose across sessions sharing a
  process.
- **Statusline "deferred" status.** v1 returned
  `{"status": "deferred"}` when a statusline arrived for an unknown
  session and produced no broadcast. v2 ingestion is purely
  append-only — every POST inserts and broadcasts.
- **WebSocket message types `session_discovered`,
  `session_ended`, `session_cleared`, `hook_event`,
  `agent_started`, `agent_stopped`, `statusline_update`,
  `limits_updated`.** Replaced by `hook_logged`,
  `statusline_logged`, `pid_death_logged`, `api_limits_logged`
  (one type per log table). Clients dispatch on `event_name` for
  the higher-level signals they want.
- **N×`session_ended` broadcast on PID death.** v1 emitted one
  `session_ended` per session in a dying process. v2 emits one
  `pid_death_logged` and lets clients fan out.
- **`SessionDetailResponse`.** Folded into the unified
  `SessionResponse` (now always carries nested arrays) and
  `ProcessResponse`.
- **`SummaryResponse.active_sessions`.** Replaced by
  `active_processes`, `active_sessions`, `active_agents`,
  `active_epochs` (the v2 summary distinguishes them).
- **Discovery loop driving session-level state mutations.** v2's
  watcher writes `claude_log_pid_deaths` rows; session/agent
  "ended" status is derived at query time.

## Added

- **`/api/v2/processes`, `/api/v2/agents`, `/api/v2/sessions/{id}/epochs`.**
  Resources for the entities v1 only addressed indirectly.
- **`/api/v2/log/{hooks|statuslines|pid-deaths|api-limits}`.** Raw
  read endpoints for each append-only log table. Replace the partial
  coverage of v1's `/api/v1/claude/sessions` and `/api/v1/claude/events`.
- **`process_id`, `epoch_id`.** Server-derived opaque ids for the
  derived process and epoch entities. On every relevant response
  and every WebSocket frame.
- **`pid`, `source_system`, `cwd` on every WebSocket frame.** v1
  carried only `session_id` and `cwd`; v2 makes routing to a
  process group possible without a REST hop.
- **`log_id` on every WebSocket frame.** Enables client-side gap
  recovery on reconnect via the raw log endpoints.
- **`include=raw` query param.** Raw payload is opt-in on derived
  event / statusline endpoints; default responses are lean.
- **`/ws/v2` endpoint.** Separate from v1's `/ws`.

## Changed shape

- **`/api/v2/sessions` returns sessions, not processes.** v1's
  `/api/v1/sessions` returned a flat session list (with cumulative
  metrics duplicated across sessions sharing a process). v2 splits
  the resources: `/api/v2/processes` returns processes,
  `/api/v2/sessions` returns sessions.
- **All timestamps are seconds.** v1's `started_at` was
  milliseconds; v2 normalizes to fractional seconds throughout.
- **Limits unavailability returns 503, not 404.** Limits not
  available is an operational state, not a missing resource.
- **`derive_state` mapping fixed to match
  [`../reference/state-transitions.md`](../reference/state-transitions.md).**
  v1 maps `Stop → idle` and orders priority `working > ready`. v2
  maps `Stop → ready` (transient until the user acknowledges) and
  orders priority `ready > working` (main session done outranks a
  background agent still ticking).

## Schema changes

- **All tables renamed and reshaped.** `claude_sessions`,
  `claude_agents`, `claude_events`, `claude_statusline`,
  `claude_costs`, `claude_limits` → `claude_log_hooks`,
  `claude_log_statuslines`, `claude_log_pid_deaths`,
  `claude_log_api_limits`. All append-only logs; no upserts.
- **No more "current state" table.** Session, process, agent, and
  epoch are derived projections over the logs.
- **Cumulative metric columns removed from session tables.** They
  were dead-write columns in late v1 anyway; v2 doesn't carry them.
- **Time normalized at ingestion.** Any payload value that arrives
  in milliseconds (e.g. statusline's `total_duration_ms`) is
  divided by 1000 at ingestion before storage.

## Implementation strategy

### Migration mechanism

**Side-by-side tables, then replay, then retire.** New
`claude_log_*` tables are created alongside the existing v1 tables
in the same SQLite database. A one-shot migration replays from
v1's `raw_payload` columns into the v2 tables (see
[`data-migration.md`](data-migration.md)). v1 tables are dropped
in the final slice.

### v1 endpoint lifecycle

v1 (`/api/v1/`, `/ws`) stays live during the slice rollout — the
service must keep working for existing clients. **No new clients
should target v1.** The final slice removes v1 endpoints and
tables together. There is no long-lived parallel-coexistence
period and no v1 feature freeze in between (whatever v1 does
today, it keeps doing until removal).

### Slicing

| Slice | Scope |
|---|---|
| A | `claude_log_hooks` + `claude_log_statuslines` + `processes` / `sessions` REST + `hook_logged` / `statusline_logged` WS. The 80% client experience. |
| B | `claude_log_pid_deaths` + watcher + `pid_death_logged` WS. |
| C | `claude_log_api_limits` + `api_limits_logged` WS. |
| D | `epochs` resource + `epoch_id` derivation. Polish; not load-bearing for the dashboard. |
| E | Remove v1 — endpoints, code, tests, `/api/v1/`, `/ws`, v1 tables. |

Each slice is independently mergeable. Slices A–D are additive;
slice E is the cutover.

### PID watcher operational details

- **Cadence.** Same as today (config-driven; default ~5s).
- **Death criterion.** No process at the PID. Image mismatch is
  *not* checked — limited value, and the source-of-truth is the
  process being gone, not its identity changing.
- **Watch set.** PIDs that appear in recent `claude_log_hooks` for
  sessions known to be alive (no `SessionEnd` hook yet, no
  `claude_log_pid_deaths` row for the same `(pid, source_system, cwd)`).
  No need to watch every pid that ever appeared.

## Compatibility

When v1 is removed (slice E), this `docs/old/` tree goes too.
