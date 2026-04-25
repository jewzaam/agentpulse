# Statusline Relay Payload

Reference: what the statusline relay
(`~/.claude/my-claude-stuff/scripts/statusline.py`) emits when POSTing to
`/statusline/claude`. Purely the wire shape — not what any consumer does
with it.

The relay script is external to this repo (lives in the user's personal
dotfiles). This doc is the pinned reference so we don't have to re-read
it.

## Upstream reference

Anthropic's statusline documentation is the authoritative source for
Claude-Code-native fields: **https://code.claude.com/docs/en/statusline**.
Anthropic does not publish a formal JSON Schema — the docs are prose
with examples. Treat the fields documented below as the working set;
new fields may appear in future Claude Code releases and will land in
`raw_payload` before the schema catches up.

## Example payload

```json
{
  "session_id": "uuid",
  "pid": 12345,
  "source_system": "host.example",
  "transcript_path": "C:\\Users\\...\\<session>.jsonl",
  "cwd": "/path/to/project",
  "workspace": {
    "project_dir": "/path/to/project",
    "current_dir": "/path/to/project",
    "added_dirs": ["..."]
  },
  "model": {
    "id": "claude-opus-4-7[1m]",
    "display_name": "Opus 4.7 (1M context)"
  },
  "version": "2.1.117",
  "cost": {
    "total_cost_usd": 5.25,
    "total_duration_ms": 120000,
    "total_api_duration_ms": 30000,
    "total_lines_added": 100,
    "total_lines_removed": 20
  },
  "context_window": {
    "total_input_tokens": 10000,
    "total_output_tokens": 50000,
    "context_window_size": 1000000,
    "current_usage": {
      "input_tokens": 5,
      "output_tokens": 200,
      "cache_creation_input_tokens": 500,
      "cache_read_input_tokens": 9000
    },
    "used_percentage": 25,
    "remaining_percentage": 75
  },
  "exceeds_200k_tokens": false,
  "rate_limits": {
    "five_hour": {"used_percentage": 3, "resets_at": 1776920400},
    "seven_day": {"used_percentage": 71, "resets_at": 1776981600}
  },
  "output_style": {"name": "default"}
}
```

## Field origin

| Source | Fields |
|---|---|
| Claude Code (native) | `session_id`, `transcript_path`, `cwd`, `workspace`, `model`, `version`, `cost`, `context_window`, `exceeds_200k_tokens`, `rate_limits`, `output_style` |
| Relay-injected | `pid`, `source_system` |

### Relay-injected fields

| Field | Type | Computation |
|---|---|---|
| `source_system` | string | `platform.node()` — hostname of the machine running Claude Code |
| `pid` | int | Claude Code's PID. Relay walks `psutil.Process(os.getpid()).parents()` and returns the first ancestor whose process name contains `"claude"`. Falls back to `os.getppid()` if psutil is unavailable or no match. |

## Semantic notes

- `cost.total_cost_usd` is **cumulative over the Claude process lifetime**,
  not per-session. It persists across `/clear` inside one process and
  resets to zero on `claude --resume` into a new process. Consequence:
  across a session_id that spans two process lifetimes (resume), the
  series is not monotonic — see the resume/reset findings in project
  memory and prior debugging sessions.
- `pid` can change within a single `session_id` if the user resumes the
  session from a new process. Each POST carries the pid that was current
  at the moment of the POST.
- `workspace.project_dir`, `workspace.current_dir`, and the top-level
  `cwd` can diverge. The statusline code that renders the "project"
  cost uses `workspace.project_dir`.
- `rate_limits.*.resets_at` is a Unix epoch *second* (integer), not an
  ISO string. Distinct from `claude_limits.raw_response.*.resets_at`
  from the OAuth API, which is an ISO-8601 string.

## Emission cadence

- Fires once per statusline render, which Claude Code triggers after
  each tool call and on some input events. Observed rate on an active
  session: roughly once per few seconds during turns, zero while idle.
- Never fires while Claude is waiting for user input between turns.
- Non-retrying on relay-side failure; a statusline post that fails is
  dropped.
