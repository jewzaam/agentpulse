# agentpulse

[![Test](https://github.com/jewzaam/agentpulse/actions/workflows/test.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/test.yml) [![Coverage](https://github.com/jewzaam/agentpulse/actions/workflows/coverage.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/coverage.yml) [![Lint](https://github.com/jewzaam/agentpulse/actions/workflows/lint.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/lint.yml) [![Format](https://github.com/jewzaam/agentpulse/actions/workflows/format.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/format.yml) [![Type Check](https://github.com/jewzaam/agentpulse/actions/workflows/typecheck.yml/badge.svg)](https://github.com/jewzaam/agentpulse/actions/workflows/typecheck.yml)
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

### Development

```bash
git clone https://github.com/jewzaam/agentpulse.git
cd agentpulse
make install-dev
```

### From Git

```bash
pip install git+https://github.com/jewzaam/agentpulse.git
```

## Usage

```bash
python -m agentpulse
```

The service starts on `127.0.0.1:17385` by default. Configure via environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `AGENTPULSE_HOST` | Bind address | `127.0.0.1` |
| `AGENTPULSE_PORT` | Bind port | `17385` |
| `AGENTPULSE_DB_PATH` | SQLite database path | `~/.claude/agentpulse/agentpulse.db` |
| `AGENTPULSE_DISCOVERY_INTERVAL` | Session scan interval (seconds) | `5` |
| `DEBUG` | Enable debug logging | `false` |

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
make check       # Run all checks (format, lint, typecheck, test, coverage)
make test        # Run tests only
make run         # Start the service (DEBUG=1 for debug logging)
```
