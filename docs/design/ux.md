# UX — Producer & Consumer Flows

End-to-end view of how data enters AgentPulse (producer side) and
how clients see it (consumer side). Companion to
[`database.md`](database.md), [`api.md`](api.md), and
[`websocket.md`](websocket.md). This doc is the "tie everything
together" page — the others are normative; this one is descriptive.

## Roles

| Role | Process | What it produces / consumes |
|---|---|---|
| **Hook relay** | `scripts/hook_relay.py`, invoked by Claude Code per hook event | Posts to `POST /hooks/claude` |
| **Statusline relay** | `~/.claude/my-claude-stuff/scripts/statusline.py`, invoked by Claude Code per statusline render | Posts to `POST /statusline/claude` |
| **PID watcher** | Internal background loop in the AgentPulse service | Writes to `claude_log_pid_deaths` |
| **Limits fetcher** | Internal background loop, opt-in via `fetch_limits` | Writes to `claude_log_api_limits` |
| **AgentPulse service** | `agentpulse-service` on `host:port` | Receives producer signals; serves consumers via REST + WS |
| **Tray app** | `agentpulse-tray` | Local consumer; one example client |
| **Other clients** | Dashboards, IDE extensions, external services | Consume REST + WS |

## Producer side

### Hook relay

Claude Code triggers the relay on every hook event (see
[`hooks-payload.md`](../reference/hooks-payload.md) for the wire
shape). The relay:

1. Reads the JSON Claude Code passes on stdin.
2. Resolves the hosting Claude PID by walking
   `psutil.Process(os.getpid()).parents()`.
3. Adds `pid` and `source_system` (hostname).
4. POSTs to `http://{host}:{port}/hooks/claude` with a 2-second
   timeout.

Failure semantics: **fire-and-forget**. A POST that fails is
dropped. Claude Code does not re-invoke the hook. This bounds the
per-event cost paid by every Claude action; data loss on transient
service unavailability is the deliberate trade.

Service-side, the endpoint inserts one row into `claude_log_hooks`
and broadcasts one `hook_logged` WebSocket frame. No upserts. No
sessions table to keep in sync. This is the single externally-
observable side effect of a hook.

### Statusline relay

Claude Code triggers the statusline relay roughly once per tool
call (see
[`statusline-payload.md`](../reference/statusline-payload.md)). The
relay forwards the full JSON to `POST /statusline/claude`, again
fire-and-forget, and then renders its own status line for the user.

Service-side, the endpoint:

1. Inserts one row into `claude_log_statuslines`. No conditional —
   even when the matching hook for the `session_id` hasn't arrived,
   the row is data and gets stored.
2. Broadcasts one `statusline_logged` WebSocket frame.

Critically, **statuslines never trigger session creation**. The
first hook for a session_id is what introduces the session into the
derived view. Until then, the statusline rows sit harmlessly in the
log; once the hook lands, derived queries pick them up
retroactively.

### PID watcher

A background loop scans the live PIDs implied by the latest hook /
statusline rows and writes a row to `claude_log_pid_deaths` when a
PID it was tracking is gone (no process at that PID, or a process
with a different image — i.e. PID reuse from our perspective).

Frequency: configured via `discovery_interval_seconds`. Writes are
append-only — repeated death observations of an already-dead
process produce no extra rows beyond the first.

This watcher provides the only signal that **isn't** derivable from
the activity logs themselves. Hooks falling silent might mean the
process died, or just that Claude is idle — the watcher disambiguates.

### Limits fetcher

When `fetch_limits: true` is set in the config, a loop fetches the
Anthropic OAuth usage-limits endpoint, normalizes ISO timestamps to
epoch seconds, and writes one row to `claude_log_api_limits`. It
also broadcasts one `api_limits_logged` WebSocket frame.

When `fetch_limits` is false, the loop is fully inert: no OAuth
token is read, no API call is made, no row is written. Clients see
404 / 503 from `/api/v2/limits` and no `api_limits_logged`
broadcasts.

### Producer principles

- **One signal, one log.** A producer writes to exactly one log
  table per signal. No producer modifies state in two places.
- **Append-only.** Producers never UPDATE. State rebuild is always
  possible by replaying the logs.
- **Self-describing rows.** Every row carries the keys needed to
  interpret it — no implicit "look up the session row to find the
  pid" step on the consumer side.
- **Time normalization at the boundary.** Whatever unit the wire
  uses (ms, ISO string), the producer's writer normalizes to
  epoch seconds before the row goes to disk.

## Consumer side

### Initial bootstrap

A new consumer wants the current state of the world.

1. **Connect WebSocket** at `/ws/v2`. Buffer incoming frames; do
   not process them yet.
2. **Fetch processes**: `GET /api/v2/processes?active=true`. Get the
   list of currently-alive processes with their summary fields
   (cost, current session, derived state).
3. **Optional drilldowns** if the client renders detail:
   - `GET /api/v2/processes/{process_id}` — sessions, epochs nested
   - `GET /api/v2/sessions/{session_id}` — agents, epochs nested
4. **Optional limits**: `GET /api/v2/limits`.
5. **Apply buffered frames + subsequent live frames**.

The "buffer then drain" pattern avoids the race where a frame
arrives between steps 2 and 5 and could otherwise be missed.

### Steady state

Two classes of consumer.

#### a. Live dashboard

Pulls bootstrap once, then applies WebSocket frames forever. The
WebSocket carries everything the dashboard needs in steady state —
no polling loop, no coarse-cadence refetch.

REST is only consulted at three boundary moments:

1. **Initial bootstrap** — see above. Loads the active set.
2. **New session discovered** — when a `hook_logged` arrives for a
   `session_id` the client hasn't seen, fetch
   `GET /api/v2/processes/{process_id}` once to pick up
   `cost_by_day` for the enclosing process. (This is the
   single-session specialization of bootstrap.) Cache it locally.
3. **Day boundary crossover** — at local midnight, refresh
   `cost_by_day` for every active process. Today's bucket rolls;
   yesterday's becomes prior-day. After midnight, apply
   `statusline_logged` deltas to the new day's bucket.

Between those moments the local cache is authoritative.
`statusline_logged` carries `cost_usd` (the running cumulative);
the client maintains "today's cost" itself by tracking deltas
from the post-midnight starting point.

Optional sanity check: on a long cadence (e.g. every 5 minutes),
`GET /api/v2/processes?active=true` and compare each process's
`last_activity_at` against the last frame seen for that
`process_id`. A drift larger than one statusline cadence suggests
missed messages — bootstrap again. This is a guardrail for
suspected drift, not a primary loop.

#### b. Periodic poller

No WebSocket. `GET /api/v2/processes?active=true` and
`GET /api/v2/summary` on a schedule. Anti-pattern for live UIs —
fine for low-frequency consumers (emails, weekly reports, billing
rollups) where real-time fidelity doesn't matter and reconnect
logic isn't worth maintaining.

### Reconnect

WebSocket connections drop. The server provides no replay.
**Reconnect by re-running bootstrap** — same procedure as initial
connect. Don't reinvent gap-fill from raw log tables; bootstrap is
already a complete snapshot.

For session discovery deltas during the disconnect window,
bootstrap will surface them naturally. Per-message history is
recoverable from the raw log endpoints
(`GET /api/v2/log/{table}?since=T`) for clients that care, but
that's a separate concern from staying live.

### Mapping wire frames to user-visible UX

Some examples to ground the abstractions.

| User-visible event | Producer wire signal | Consumer wire signal |
|---|---|---|
| User typed a message into Claude | `UserPromptSubmit` hook | `hook_logged` with `event_name=UserPromptSubmit` |
| Claude is running a Bash command | `PreToolUse` hook with `tool_name=Bash` | `hook_logged` |
| Claude is asking permission to run something | `PermissionRequest` hook | `hook_logged` — UI surfaces permission_required state |
| Subagent started | `SubagentStart` hook | `hook_logged` with `event_name=SubagentStart` and `agent_id`, `agent_type` |
| Claude finished a turn | `Stop` hook | `hook_logged` with `event_name=Stop` — UI flips to idle |
| Cost ticked up | Statusline render | `statusline_logged` with new `cost_usd` |
| User ran `/clear` inside Claude | Subsequent hooks now arrive with new `session_id`, same `pid` | First `hook_logged` for the new session — UI adds a new session under the existing process |
| User ran `claude --resume` from a different terminal | Hooks now arrive with same `session_id`, new `pid` | `hook_logged` with new `pid` — UI starts a new epoch under the same session |
| Claude process killed (terminal closed) | Watcher writes `pid_death_logged` | `pid_death_logged` — UI fans out: the process and all its sessions / agents end |
| User clicked "Reset" in dashboard for a stuck session | Dashboard calls `POST /api/v2/sessions/{id}/clear` | Dashboard receives the response; all clients receive `hook_logged` with `event_name=clear_state` |
| Limits ticked over to a new bucket | Fetcher pulls new snapshot | `api_limits_logged` |

### What consumers should not do

- **Do not aggregate `cost_usd` across statusline_logged frames.**
  It is process-cumulative. Replace, don't sum.
- **Do not rely on `event_name` to enumerate cleanup actions.**
  When `pid_death_logged` arrives, the process and all its sessions
  / agents are gone — even if no `SessionEnd` was ever emitted.
- **Do not treat `hook_logged` order as authoritative for "which
  session is current."** The current session within a process is
  whichever session has the most recent `received_at`. A late-
  arriving hook for an earlier session does not "switch back."
- **Do not expect `process_id` to be portable across deployments.**
  It is a hash of `(source_system, pid, cwd, earliest_received_at)`
  — stable within a deployment, meaningless outside.

## End-to-end flow walkthrough

### Scenario: user opens VS Code, runs Claude, types a prompt, gets a response

1. **VS Code launches Claude Code as a subprocess.**
2. Claude fires `SessionStart` → relay POSTs to `/hooks/claude` →
   service inserts one row in `claude_log_hooks`, broadcasts
   `hook_logged{event_name: SessionStart}`.
3. User types a message → `UserPromptSubmit` hook → another row,
   another broadcast.
4. Claude calls Read tool → `PreToolUse{tool_name: Read}` and
   `PostToolUse{tool_name: Read}` → two rows, two broadcasts.
5. Claude renders statusline → relay POSTs `/statusline/claude` →
   `claude_log_statuslines` row, `statusline_logged` broadcast.
6. Claude finishes turn → `Stop` hook → row, broadcast.
7. User runs `/clear` inside Claude. Next prompt arrives with a
   new `session_id`. The dashboard sees the first `hook_logged` for
   the new session and adds it under the same `process_id`.
8. User closes the terminal. Claude exits. `SessionEnd` may or may
   not fire (depends on signal). Within a few seconds, the PID
   watcher notices the PID is gone and writes
   `claude_log_pid_deaths`. Service broadcasts `pid_death_logged`.
9. Dashboard fans out: marks the process dead, all sessions ended,
   all agents ended.

Every step is one log write and one WebSocket frame. The data model
and the wire shape stay 1:1.

## Producer/consumer contract summary

| Concern | Producer obligation | Consumer obligation |
|---|---|---|
| Identity on each row | Carry `(session_id, pid, source_system, cwd, agent_id?)` | Use it to route messages without a REST lookup |
| Time format | Normalize to epoch seconds at ingest | Treat all times as seconds |
| Order | Append-only, write order matches `id` order | Ordering on `id` is authoritative; `received_at` ties may exist |
| Failure | Fire-and-forget; drop on error | No retransmission expected; rely on REST gap-fetch on reconnect |
| Schema evolution | New fields land in `raw_payload` first | Tolerate unknown `event_name` / `tool_name` / etc.; do not enum-validate |
| Side effects | Exactly one log row per producer signal | Exactly one displayed change per consumer expectation |
