# agentpulse

[![test-unit](https://github.com/jewzaam/agentpulse/actions/workflows/test-unit.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/test-unit.yml) [![test-coverage](https://github.com/jewzaam/agentpulse/actions/workflows/test-coverage.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/test-coverage.yml) [![test-lint](https://github.com/jewzaam/agentpulse/actions/workflows/test-lint.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/test-lint.yml) [![test-format](https://github.com/jewzaam/agentpulse/actions/workflows/test-format.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/test-format.yml) [![test-typecheck](https://github.com/jewzaam/agentpulse/actions/workflows/test-typecheck.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/test-typecheck.yml)
[![Python 3.12+](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/) [![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

A backend service that collects agent state from Claude Code sessions and exposes it for querying via REST and WebSocket.

## Overview

AgentPulse monitors Claude Code sessions through hook events and session file discovery, persists state to SQLite, and serves it through a FastAPI application.

- Receives real-time hook events (tool calls, state transitions, agent lifecycle)
- Discovers sessions from `~/.claude/sessions/*.json`
- Tracks subagent state per session
- Exposes normalized REST API and platform-specific raw endpoints
- Streams events via WebSocket

## Installation

### 1. Install

Core service (no GUI dependencies):

```bash
pipx install git+https://github.com/jewzaam/agentpulse.git
```

With the optional tray icon:

```bash
pipx install "agentpulse[tray] @ git+https://github.com/jewzaam/agentpulse.git"
```

For local development:

```bash
git clone https://github.com/jewzaam/agentpulse.git
cd agentpulse
make install-dev
```

### 2. Configure

```bash
agentpulse init
```

Creates `~/.claude/agentpulse/config.json` from the bundled example. Edit it to
change `host`, `port`, or `fetch_limits`. The service exits with an error if this
file is missing.

### 3. Install Claude Code hooks

```bash
make install-hooks
```

Copies the relay script to `~/.claude/agentpulse/scripts/` and deep-merges the
hook configuration into `~/.claude/settings.json`. Idempotent.

### 4. Enable auto-start on login

```bash
make install-autostart-service
```

- **Linux:** installs `~/.config/systemd/user/agentpulse.service` and runs
  `systemctl --user enable --now agentpulse`.
- **Windows:** creates `AgentPulse.lnk` in your Startup folder.

### 5. (Optional) Enable tray icon auto-start

Requires the `[tray]` extra from step 1.

```bash
make install-autostart-tray
```

- **Linux:** installs `~/.config/autostart/agentpulse-tray.desktop`.
- **Windows:** creates `AgentPulseTray.lnk` in your Startup folder.

### Shortcut: install everything

From a checkout, after `agentpulse init`:

```bash
make install   # = install-pipx + install-hooks + install-autostart-service
```

Tray autostart is opt-in. Add it with `make install-autostart-tray`.

### Manual launch (no autostart)

```bash
make run-service   # service only, foreground
make run-tray      # tray only, foreground
make run           # both (Ctrl-C exits both)
```

### Uninstall

```bash
make uninstall                    # autostart off (service + tray) + pipx uninstall
rm -rf ~/.claude/agentpulse/
```

(Hook entries in `~/.claude/settings.json` must be removed manually.)

## Usage

```bash
python -m agentpulse --config ~/.claude/agentpulse/config.json
```

All configuration is via the config file (`--config` is required). See `config.example.json` for the schema:

| Key | Description | Default |
|-----|-------------|---------|
| `host` | Bind address | `127.0.0.1` |
| `port` | Bind port | `17385` |
| `db_path` | SQLite database path | `~/.claude/agentpulse/agentpulse.db` |
| `log_file` | Rotating log file path | `~/.claude/agentpulse/agentpulse.log` |
| `log_level` | Logging level (DEBUG, INFO, WARNING, ERROR) | `INFO` |
| `discovery_interval_seconds` | Session scan interval | `5` |

### API Endpoints

| Endpoint | Description |
|----------|-------------|
| `POST /hooks/claude` | Receive Claude Code hook events |
| `GET /api/v1/sessions` | List all sessions (normalized) |
| `GET /api/v1/sessions/{id}` | Session detail with agents |
| `GET /api/v1/sessions/{id}/events` | Paginated event history |
| `GET /api/v1/summary` | Aggregate stats |
| `GET /api/v1/claude/sessions` | Raw Claude session data |
| `GET /api/v1/claude/events` | Raw Claude events with filters |
| `ws://host:port/ws` | Real-time event stream |

## Development

```bash
make check         # Full quality gate: test-format/lint/typecheck/unit/coverage
make test-unit     # Run pytest only
make run-service   # Start the service in the foreground
```

Run `make help` for the full list of targets.
