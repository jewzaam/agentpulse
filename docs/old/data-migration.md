# Data Migration — v1 → v2

How to rebuild the v2 `claude_log_*` tables from existing v1 data.
Goes away when the migration is done; this is a transitional doc.

## Data span

The agentpulse SQLite database covers from **2026-04-13** to
present. v1 has roughly two weeks of high-fidelity hook +
statusline + limits data. The migration uses this as the
authoritative source.

There is also an older external archive at
`~/.claude/my-claude-stuff-data/` going back to mid-March 2026 —
about a month of pre-agentpulse history. Most of it is too lossy to
ingest meaningfully (no pid, no tool detail, no statusline
cadence). The one slice worth considering is
`session-tracker/`'s daily snapshots if you want a populated
`cost_by_day` for old projects in a "historical costs" report.
That's an optional appendix at the end of this doc; the main
migration ignores it.

## What we're moving

The v2 schema in [`../design/database.md`](../design/database.md)
is purely append-only logs. Every wire-sourced row preserves a
`raw_payload` (or `raw_response`). v1 already does the same:

- `claude_events.raw_payload` — full hook JSON
- `claude_statusline.raw_payload` — full statusline JSON
- `claude_limits.raw_response` — full OAuth limits API response

These three columns are the authoritative input — everything else
can be re-derived from them at ingest time.

## Target tables

`claude_log_hooks`, `claude_log_statuslines`, `claude_log_pid_deaths`,
`claude_log_api_limits`. Schemas in
[`../design/database.md`](../design/database.md). All fields below
refer to those columns.

## Mapping

### `claude_log_hooks` ← v1 `claude_events`

One row per source row. The hook table's `raw_payload` carries
everything needed; extracted columns are derived from it.

**Skip rows whose `raw_payload` does not contain `pid`.** v1's
relay didn't inject `pid` (or `source_system`) until late on the
first day of agentpulse use (2026-04-13). 14 sessions / 585 hook
events from that window are pid-less; carrying them forward would
just produce orphans in v2's process derivation. They're dropped
during replay. This is a one-time loss bounded to day one.

| target column | source |
|---|---|
| `id` | new autoincrement |
| `received_at` | `claude_events.received_at` |
| `received_by` | `"hook-endpoint-v1-replay"` |
| `session_id` | `claude_events.session_id` |
| `pid` | `raw_payload.pid` |
| `source_system` | `raw_payload.source_system` |
| `cwd` | `claude_events.cwd` (or `raw_payload.cwd`) |
| `agent_id` | `claude_events.agent_id` |
| `event_name` | `claude_events.event_name` |
| `tool_name` | `claude_events.tool_name` |
| `agent_type` | `raw_payload.agent_type` (null when absent) |
| `permission_mode` | `raw_payload.permission_mode` |
| `entrypoint` | look up from `claude_sessions.entrypoint` for that `session_id` |
| `claude_version` | `null` (v1 hook payloads don't carry it) |
| `raw_payload` | `claude_events.raw_payload` verbatim |

Replay all rows ordered by `received_at, id` so the new log_id
ordering tracks original chronology.

### `claude_log_statuslines` ← v1 `claude_statusline`

**Skip rows whose `raw_payload` does not contain `pid`.** Same
day-one window as hooks (~22 sessions / 1,151 statusline rows
predate the relay's pid injection). Drop on replay.

| target column | source |
|---|---|
| `received_at` | `claude_statusline.received_at` |
| `received_by` | `"statusline-endpoint-v1-replay"` |
| `session_id` | `claude_statusline.session_id` |
| `pid` | `raw_payload.pid` |
| `source_system` | `raw_payload.source_system` |
| `cwd` | `raw_payload.cwd` |
| `project_dir` | `raw_payload.workspace.project_dir` |
| `model_id` | `raw_payload.model.id` |
| `model_name` | `raw_payload.model.display_name` (or v1 `model_name` column) |
| `claude_version` | `raw_payload.version` |
| `cost_usd` | `claude_statusline.cost_usd` |
| `total_input_tokens` / `total_output_tokens` | corresponding columns |
| `context_used_pct`, `context_window_size` | corresponding columns |
| `cache_read_tokens`, `cache_creation_tokens` | corresponding columns |
| `duration_seconds` | `claude_statusline.duration_ms / 1000.0` |
| `api_duration_seconds` | `claude_statusline.api_duration_ms / 1000.0` |
| `lines_added`, `lines_removed` | corresponding columns |
| `exceeds_200k_tokens` | `raw_payload.exceeds_200k_tokens` |
| `raw_payload` | `claude_statusline.raw_payload` verbatim |

### `claude_log_pid_deaths` ← synthesized from v1 `claude_sessions`

No v1 table for this. Synthesize:

- For each `claude_sessions` row where `pid_alive = 0`:
  - Write one `claude_log_pid_deaths` row with
    `pid = sessions.pid`, `source_system = sessions.source_system`,
    `cwd = sessions.cwd`, `observed_at = sessions.ended_at`,
    `observed_by = "pid-watcher-v1-replay"`.
- Skip rows where `ended_at` is null or `pid_alive = 1`.

Caveats:
- v1 collapsed N sessions sharing one process into N "dead" flags
  each carrying the same death moment. This produces N death rows
  for what was really one process death. Acceptable — v2 queries
  group on `(pid, source_system, cwd)` and dedupe by composite +
  timestamp window.
- Sessions that ended via `SessionEnd` rather than PID death also
  have `ended_at` set; those should NOT produce a death row. Filter
  on `pid_alive = 0` (the discovery loop's signal) rather than on
  `ended_at` alone.

### `claude_log_api_limits` ← v1 `claude_limits`

One row per source row.

| target column | source |
|---|---|
| `received_at` | `claude_limits.fetched_at` |
| `received_by` | `"limits-fetcher-v1-replay"` |
| `five_hour_utilization` | `raw_response.five_hour.utilization` |
| `five_hour_resets_at` | `raw_response.five_hour.resets_at` (ISO) → epoch seconds |
| `seven_day_utilization` | `raw_response.seven_day.utilization` |
| `seven_day_resets_at` | `raw_response.seven_day.resets_at` (ISO) → epoch seconds |
| `raw_response` | `claude_limits.raw_response` verbatim |

## Time normalization

v2 stores epoch seconds (REAL with fractional precision)
everywhere. Conversions:

- `claude_statusline.duration_ms` / `api_duration_ms` →
  `seconds = ms / 1000.0`
- `claude_sessions.started_at` (milliseconds int) — unused at
  ingest time (the session row goes away in v2; `started_at` is
  derived). If you need it for a sanity check, divide.
- Limits API `resets_at` ISO strings →
  `datetime.fromisoformat(...).timestamp()`.

## Idempotency

The migration script must be re-runnable. Two approaches:

1. **Drop-and-replay.** Empty the v2 tables before each run.
   Simple and reliable; the migration runs in seconds for typical
   data volumes.
2. **High-watermark replay.** Track a per-source watermark
   (`v1.claude_events.id`) and only ingest rows past the watermark.
   More complex; only worth it if migration time becomes a real
   concern.

Drop-and-replay is the recommended default.

## Order of operations

```
1. Drop and recreate v2 tables (per design/database.md).
2. Replay v1 claude_events       → claude_log_hooks
3. Replay v1 claude_statusline   → claude_log_statuslines
4. Replay v1 claude_limits       → claude_log_api_limits
5. Synthesize from claude_sessions → claude_log_pid_deaths
```

## Validation

After migration:

- **Row counts.**
  - `count(claude_log_hooks where received_by = 'hook-endpoint-v1-replay')`
    should equal v1 `count(claude_events)` minus any rows whose
    `raw_payload` failed to parse.
  - Same check for statuslines and limits.
- **Sample integrity.**
  - Pick 5 random source rows; confirm the v2 row carries the
    same `raw_payload` byte-for-byte.
- **Process grouping.**
  - For a known-active session, verify the v2 process derivation
    returns the same `(pid, source_system, cwd)` groups it had on
    v1 list-by-process snapshots.
- **Cost reconciliation.**
  - For one session, take v1's last `claude_statusline.cost_usd`;
    it should equal the v2-derived cumulative cost for the same
    process group.

## What's lost / approximate

- **Anything before 2026-04-13.** v1 didn't exist. Older Claude
  transcripts under `~/.claude/projects/` are still present, but
  they're not part of the v2 data model.
- **PID deaths before discovery loop ran reliably.** Sessions that
  died early in v1's history may not have a `pid_alive=0` flip; no
  death row is synthesized for those. Acceptable loss.

The migration won't reconstruct what was never captured.

## Optional appendix — historical daily costs from `session-tracker/`

`~/.claude/my-claude-stuff-data/session-tracker/YYYY-MM-DD/<session_id>.json`
holds one statusline-shaped snapshot per session per day, going
back to mid-March 2026. Each file carries `pid`, `source_system`,
`cwd`, `cost`, model, context — enough to populate
`claude_log_statuslines` for sessions that predate v1.

The point of doing this: a "what did each project cost in March?"
report. The cost trajectory is one point per day rather than a
real curve, but the daily totals come out right because v2's
`cost_by_day` derivation uses `MAX(cost_usd)` per day.

Mapping:

| target column | source |
|---|---|
| `received_at` | the file's mtime, **or** the day's local-midnight epoch + 23:59 if mtime is unreliable. Day directory name is the calendar day. |
| `received_by` | `"session-tracker-replay"` |
| `session_id` | `session_id` field |
| `pid` | `pid` field |
| `source_system` | `source_system` field |
| `cwd` | `cwd` field |
| `project_dir` | `workspace.project_dir` |
| `model_id` / `model_name` | `model.id` / `model.display_name` |
| `claude_version` | `version` |
| `cost_usd` | `cost.total_cost_usd` |
| `total_input_tokens` / `total_output_tokens` | `context_window.total_input_tokens` / `total_output_tokens` |
| `context_used_pct` | `context_window.used_percentage` |
| `context_window_size` | `context_window.context_window_size` |
| `cache_read_tokens` | `context_window.current_usage.cache_read_input_tokens` |
| `cache_creation_tokens` | `context_window.current_usage.cache_creation_input_tokens` |
| `duration_seconds` | `cost.total_duration_ms / 1000.0` |
| `api_duration_seconds` | `cost.total_api_duration_ms / 1000.0` |
| `lines_added` / `lines_removed` | `cost.total_lines_added` / `total_lines_removed` |
| `exceeds_200k_tokens` | `exceeds_200k_tokens` |
| `raw_payload` | the file content verbatim |

Caveats:
- One snapshot per session per day, so cost trajectory is sparse.
  Daily totals are correct; intra-day curves aren't recoverable.
- `_prior_cost` in the file is internal scaffolding from a
  now-removed computation; ignore it.
- Only replay archive snapshots for sessions absent from v1.
- Some session-tracker files predate `source_system` injection.
  Treat those rows as single-machine; filter them out if
  multi-machine attribution matters.
- No corresponding hooks for these sessions, so the derived view
  shows them only in the cost rollup. The session lists won't show
  meaningful event activity. That's the point — this is a
  historical-cost extension, not a session-replay.

The `prompt-log/` directory in the same archive is **not**
ingested. It would synthesize fake `UserPromptSubmit`/`Stop` hook
rows with no pid, no tool detail, no agent detail — looks active
in the UI but lies about everything except "a session existed."
Not worth the complexity.

## When to delete this doc

When v1 is fully removed from the codebase and no production
database needs migrating. At that point, this whole `docs/old/`
tree goes too.
