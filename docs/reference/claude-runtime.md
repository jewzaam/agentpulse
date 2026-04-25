# Claude Code Runtime Model

Reference: how Claude Code structures processes, sessions, and the
relationship between them. This is upstream behavior, not anything
AgentPulse defines — it's the substrate the data model and the API
sit on top of.

Pinned here so the API, WebSocket, and database docs can refer to a
single description of these concepts instead of re-explaining them.

## Upstream references

- Hooks: **https://code.claude.com/docs/en/hooks**
- Statusline: **https://code.claude.com/docs/en/statusline**

Anthropic does not publish formal schemas for either — the docs are
prose with examples. The wire shapes AgentPulse observes are pinned
in [`hooks-payload.md`](hooks-payload.md) and
[`statusline-payload.md`](statusline-payload.md).

## Concepts

### Process

An OS-level Claude Code instance — the process that the user
launched (typically via `claude` or via an IDE extension that
spawns it). Identified externally by:

- `pid` — operating-system process id
- `source_system` — hostname of the machine running it
- `cwd` — current working directory when the hook fired

Claude Code does not expose its own process id; AgentPulse derives
it on the relay side by walking parents from the hook script's PID
until it finds an ancestor whose name contains `"claude"`.

### Session

A logical conversation, identified by `session_id` (a UUID Claude
Code mints). Has a transcript file under
`~/.claude/projects/<project-slug>/<session_id>.jsonl`. A user
sees one session at a time inside a Claude Code process.

### `/clear` — new session, same process

Inside a running Claude Code process, the user types `/clear`.
Claude resets the conversation context and **mints a new
`session_id`**. The OS process keeps running with the same `pid`.
A single process therefore hosts a sequence of sessions over its
lifetime — every `/clear` adds one.

### `claude --resume <session_id>` — same session, new process

The user launches a fresh `claude --resume <id>` from a new
terminal (often the next day). Claude Code starts a **new OS
process** with a new `pid` but reattaches to the existing
`session_id`. Subsequent hook events for the resumed session carry
the new `pid` even though `session_id` is unchanged.

A single `session_id` can therefore appear under multiple processes
over time — the boundaries are pid changes within the same
session_id stream.

### Cumulative metrics are process-scoped

The statusline payload's `cost`, `total_input_tokens`,
`total_output_tokens`, `total_lines_added`, `total_lines_removed`
are **cumulative over the Claude process lifetime**. They:

- persist across `/clear` (new session, but same process keeps the
  counters running)
- reset on `claude --resume` (new process starts at zero)

For any single process, the series is monotonically non-decreasing.
For a single `session_id` that spans multiple processes (resume),
the series is **not** monotonic — the counter resets at each new
process boundary.

### Per-session fields

These come from hooks and reset (or are absent) when a new session
begins:

- `last_event`, `last_tool` — derived from the most recent hook
- `context_used_pct`, `model_name` — from the statusline payload's
  per-session view
- agents — `SubagentStart` / `SubagentStop` are scoped to the
  session that introduced them; `/clear` ends them implicitly
  because the session itself ends

### Agent

A subagent invocation within a session. Introduced by a
`SubagentStart` hook with an `agent_id` and `agent_type`; closed by
a `SubagentStop` hook for the same `agent_id`. Multiple agents can
run sequentially within one session; an agent does not outlive its
parent session.

### Interrupts do not fire a hook

When the user interrupts Claude mid-turn (typically by pressing
ESC, or otherwise cancelling a running tool call), **no hook event
is emitted**. The session is left in whatever state the most recent
hook described — often a `PreToolUse` that never paired with a
`PostToolUse`. From the outside, the session looks "stuck" running
a tool that actually ended.

This is upstream behavior, not a bug in any AgentPulse component.
The data model has no way to derive an interrupt from the hook
stream — there is nothing to derive from. AgentPulse's manual
`POST .../clear` endpoint exists specifically to cover this gap:
when the user knows an interrupt happened, the dashboard can ask
the service to insert a synthetic `clear_state` row so derived
state stops showing the stale tool. Without the manual nudge, the
"working" state persists until the next real hook arrives.

If Anthropic later adds an interrupt hook event, this section
should be revisited — the synthetic `clear_state` workaround would
become unnecessary.

## Implications for AgentPulse

These are the runtime facts the AgentPulse design has to absorb:

1. **A process is the natural unit for cost.** A view that shows
   "cost per session" double-counts when one process hosted several
   sessions, and produces nonsense after a resume because the
   counter reset. Cost belongs on the process, not the session.
2. **Process identity needs a time-window discriminator.** The OS
   reuses PIDs. Two distinct Claude processes with the same
   `(pid, source_system, cwd)` tuple but separated in time are
   genuinely different processes. The data model handles this by
   anchoring on the earliest event time.
3. **Session identity is stable across processes.** For tracing a
   single conversation through resumes, key on `session_id`. For
   tracking cost / process lifecycle, key on the process tuple.
4. **The pid stream within a session_id is the resume signal.**
   When `pid` changes within a session_id, the user resumed the
   session into a new process. AgentPulse calls each `(session_id,
   pid)` span an **epoch**.
