# Test Plan

## Overview

**Project:** agentpulse
**Primary functionality:** Backend service collecting Claude Code agent state via hooks and session discovery, exposing it through REST and WebSocket APIs.

## Testing Philosophy

This project follows the [Testing Standards](https://github.com/jewzaam/standards/blob/main/python/testing.md).

Key testing principles for this project:

- Safety guards in conftest.py block real subprocess, filesystem writes, and HTTP
- All database tests use isolated tmp_path SQLite instances
- API tests use FastAPI TestClient with routers wired directly (no lifespan/background tasks)
- WebSocket tests verify broadcast behavior through hook event integration

## Test Categories

### Unit Tests

Tests for isolated functions with mocked dependencies.

| Module | Function | Test Coverage | Notes |
|--------|----------|---------------|-------|
| `config.py` | `Settings`, `get_settings`, `reset_settings` | Defaults, env var override, singleton behavior | |
| `platforms/claude/models.py` | `derive_state()` | All event/tool combinations from claude-dashboard mapping | |
| `platforms/claude/models.py` | `compute_effective_state()` | Priority ordering across session + agents | |
| `platforms/claude/models.py` | `ClaudeHookPayload` | Pydantic validation, defaults | |
| `platforms/claude/discovery.py` | `discover_sessions()` | File parsing, malformed JSON, missing fields | |
| `platforms/claude/discovery.py` | `validate_pid()` | Alive/dead/non-claude processes (psutil mocked) | |
| `platforms/claude/discovery.py` | `detect_branch()` | Normal branch, detached HEAD, non-git | |
| `websocket.py` | `ConnectionManager` | Broadcast, dead connection cleanup, disconnect | |

### Integration Tests

Tests for multiple components working together.

| Workflow | Components | Test Coverage | Notes |
|----------|------------|---------------|-------|
| Hook ingestion | hooks.py + schema.py + db.py | Payload → DB write → event log append | Via TestClient POST |
| Agent lifecycle | hooks.py + schema.py | Create on first event, end on SubagentStop | |
| Session discovery tick | discovery.py + schema.py + db.py | File scan → DB upsert → dead PID detection | Mocked psutil |
| REST API queries | api/sessions.py + schema.py | Seed DB → query → response shape + derived_state | |
| WebSocket broadcast | websocket.py + hooks.py | Hook POST → WS client receives message | |

## Untested Areas

| Area | Reason Not Tested |
|------|-------------------|
| `__main__.py` | Entry point calls uvicorn; tested manually |
| `app.py` lifespan | Background discovery loop; tested via component tests, not end-to-end lifespan |
| Real PID validation | Requires running Claude processes; mocked in tests |
| Real session file discovery | Requires `~/.claude/sessions/`; uses tmp_path mock files |

## Bug Fix Testing Protocol

All bug fixes to existing functionality **must** follow TDD:

1. Write a failing test that exposes the bug
2. Verify the test fails before implementing the fix
3. Implement the fix
4. Verify the test passes
5. Verify reverting the fix causes the test to fail again
6. Commit test and fix together with issue reference

### Regression Tests

| Issue | Test | Description |
|-------|------|-------------|
| (none yet) | | |

## Coverage Goals

**Target:** 80%+ line coverage

**Current:** 94%

**Philosophy:** Coverage measures completeness, not quality. A test that executes code without meaningful assertions provides no value. Focus on testing behavior, not implementation details.

## Running Tests

```bash
make test           # Run all tests
make coverage       # Run with coverage (80% threshold)
make test-verbose   # Run with verbose output
```

## Test Data

Test data is:
- Generated programmatically in fixtures (mock session JSON files, git HEAD files)
- No static fixtures directory needed
- All files created in `tmp_path` for automatic cleanup

**No Git LFS** - all test data must be small or generated.

## Maintenance

When modifying this project:

1. **Adding features**: Add tests for new functionality after implementation
2. **Fixing bugs**: Follow TDD protocol above (test first, then fix)
3. **Refactoring**: Existing tests should pass without modification (behavior unchanged)
4. **Removing features**: Remove associated tests

## Changelog

| Date | Change | Rationale |
|------|--------|-----------|
| 2026-04-12 | Initial test plan | Project creation |
