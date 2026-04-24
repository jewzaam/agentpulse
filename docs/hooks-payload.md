# Hook Relay Payload

Reference: what the hook relay (`scripts/hook_relay.py`) emits when
POSTing to `/hooks/claude`. Purely the wire shape — not what any
consumer does with it.

The relay script is in this repo at [`scripts/hook_relay.py`](../scripts/hook_relay.py).
Claude Code invokes it as a hook command; the relay reads Claude's hook
JSON on stdin, enriches it, and POSTs.

## Upstream reference

Anthropic's hook documentation is the authoritative source for
Claude-Code-native fields and the catalog of event names:
**https://code.claude.com/docs/en/hooks**. Anthropic does not publish a
formal JSON Schema — the docs are prose with examples.

The event list below reflects what we've observed in the wild. Anthropic
documents **additional events** we haven't yet profiled (`InstructionsLoaded`,
`UserPromptExpansion`, `PermissionDenied`, `PostToolBatch`,
`TaskCreated`/`TaskCompleted`, `TeammateIdle`, `CwdChanged`, `FileChanged`,
`WorktreeCreate`/`WorktreeRemove`, `PreCompact`/`PostCompact`,
`Elicitation`/`ElicitationResult`, `ConfigChange`, `SessionStart`
variants). Treat the table as a working subset; the authoritative catalog
is Anthropic's.

## Example payload (PreToolUse for Bash)

```json
{
  "session_id": "uuid",
  "pid": 12345,
  "source_system": "host.example",
  "transcript_path": "C:\\Users\\...\\<session>.jsonl",
  "cwd": "/path/to/project",
  "permission_mode": "acceptEdits",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "git status",
    "description": "Check working tree state"
  }
}
```

Example payload (PostToolUse adds `tool_response`):

```json
{
  "session_id": "uuid",
  "pid": 12345,
  "source_system": "host.example",
  "transcript_path": "...",
  "cwd": "/path/to/project",
  "permission_mode": "acceptEdits",
  "hook_event_name": "PostToolUse",
  "tool_name": "Bash",
  "tool_input": {"command": "git status", "description": "..."},
  "tool_response": {
    "stdout": "...",
    "stderr": "",
    "interrupted": false,
    "isImage": false,
    "noOutputExpected": false
  },
  "tool_use_id": "toolu_01TJAaH..."
}
```

Example payload (Stop adds `stop_hook_active`, `last_assistant_message`):

```json
{
  "session_id": "uuid",
  "pid": 12345,
  "source_system": "host.example",
  "transcript_path": "...",
  "cwd": "/path/to/project",
  "permission_mode": "acceptEdits",
  "hook_event_name": "Stop",
  "stop_hook_active": false,
  "last_assistant_message": "Committed as `664e805`."
}
```

## Field origin

| Source | Fields |
|---|---|
| Claude Code (native) | `session_id`, `transcript_path`, `cwd`, `permission_mode`, `hook_event_name`, plus event-specific fields (see below) |
| Relay-injected | `pid`, `source_system` |

### Relay-injected fields

| Field | Type | Computation |
|---|---|---|
| `source_system` | string | `platform.node()` — hostname of the machine running Claude Code |
| `pid` | int | Claude Code's PID. Relay walks `psutil.Process(os.getpid()).parents()` and returns the first ancestor whose process name contains `"claude"`. Falls back to `os.getppid()` if psutil is unavailable or no ancestor matches. |

## Event-specific fields

All hook events carry the base fields (`session_id`, `transcript_path`,
`cwd`, `permission_mode`, `hook_event_name`, plus the relay-injected
`pid` and `source_system`). Individual events add:

| `hook_event_name` | Additional fields |
|---|---|
| `SessionStart` | `hookSpecificOutput` (hook chain output, sometimes empty) |
| `SessionEnd` | — (no extra fields beyond the base) |
| `UserPromptSubmit` | `prompt` (the user's message text) |
| `PreToolUse` | `tool_name`, `tool_input`, `tool_use_id` |
| `PostToolUse` | `tool_name`, `tool_input`, `tool_response`, `tool_use_id` |
| `PostToolUseFailure` | same as `PostToolUse` plus failure detail inside `tool_response` |
| `PermissionRequest` | `tool_name`, `tool_input`, `tool_use_id` |
| `Notification` | `message` |
| `Stop` | `stop_hook_active`, `last_assistant_message` |
| `StopFailure` | same as `Stop` plus failure detail |
| `SubagentStart` | `agent_id`, `agent_type`, `tool_input` (the delegated task) |
| `SubagentStop` | `agent_id`, `agent_type` |

Within subagent-scoped tool events (`PreToolUse`/`PostToolUse` fired by
a subagent), `agent_id` and `agent_type` also appear alongside the
tool fields.

## Semantic notes

- `pid` is the Claude Code CLI's process ID **at the moment the hook
  fired**. When a session is resumed (`claude --resume <session_id>`)
  from a new shell, the PID in the payload changes even though
  `session_id` is unchanged. Each POST carries the current pid.
- `cwd` is Claude's current working directory. Normally matches the
  shell where `claude` was launched; resume from a different shell can
  change it.
- `transcript_path` points to Claude's per-session JSONL transcript
  under `~/.claude/projects/<project-slug>/<session_id>.jsonl`. The
  file is authoritative for session content; agentpulse treats it as
  external.
- `tool_input` and `tool_response` shapes vary per tool. For `Bash`:
  `{"command", "description"}` in; `{"stdout", "stderr", "interrupted",
  "isImage", "noOutputExpected"}` out. For other tools, the shape
  follows Claude's tool spec.
- `permission_mode` values observed: `default`, `acceptEdits`, `plan`,
  `bypassPermissions`. Claude Code may add more.

## Emission cadence

- Fires once per hook event. Tool events fire before and after each
  tool call. `UserPromptSubmit` fires when the user sends a message.
  `Stop` fires when Claude finishes a turn. `SubagentStart`/`Stop`
  bracket each subagent invocation.
- Non-retrying on relay-side failure; a hook POST that fails is
  dropped. Claude Code itself does not re-invoke the hook.
- Relay timeout: 2 seconds per POST.
