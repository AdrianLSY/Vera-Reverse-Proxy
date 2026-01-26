This is a monorepo containing the Vera-Stack dynamic reverse proxy system with WebSocket tunneling.

**Always refer to the `CLAUDE.md` file for a high-level system overview before starting work on this project.**

**After completing any task, review and update `README.md`, `AGENTS.md`, and `CLAUDE.md` as necessary to keep documentation current.**

## Project Guidelines

- This is a **monorepo with git submodules** - component code lives in submodules, not the root
- Work directly in submodules (`Plugboard/` or `Telephone/`) for component changes
- Root repository is for orchestration, CI/CD, and cross-cutting documentation
- Run component-specific precommit checks before committing:
  - **Plugboard:** `mix precommit`
  - **Telephone:** `make precommit`

## Submodule Guidelines

### General Rules

- **Never commit directly to the parent repo's submodule pointer** unless updating the reference
- **Always work inside the submodule directory** when making component changes
- Submodule commits are independent - push to the submodule repo first
- Parent repo reference updates are automated via CI/CD

### Keeping Submodules in Sync

```bash
# Fetch latest for all submodules
git submodule update --remote

# If you see "detached HEAD" in a submodule, that's normal
# To make changes, create a branch first:
cd Plugboard
git checkout main
git pull origin main
```

### Making Changes Across Components

When a change requires modifications to both Plugboard and Telephone:

1. **Plan the interface first** - agree on message formats, events, and protocols
2. **Update Plugboard** - implement server-side changes, push to Plugboard repo
3. **Update Telephone** - implement client-side changes, push to Telephone repo
4. **Test integration** - run both components together locally
5. **Document** - update relevant CLAUDE.md/AGENTS.md files

## Integration Patterns

### Communication Protocol

Plugboard and Telephone communicate via Phoenix Channels over WebSocket:

| Event | Direction | Purpose |
|-------|-----------|---------|
| `phx_join` | Telephone -> Plugboard | Join channel with JWT token |
| `proxy_req` | Plugboard -> Telephone | Forward HTTP request |
| `proxy_res` | Telephone -> Plugboard | Return HTTP response |
| `ws_check` | Plugboard -> Telephone | Verify WebSocket support |
| `ws_check_result` | Telephone -> Plugboard | WebSocket support confirmation |
| `ws_frame` | Bidirectional | WebSocket frame relay |
| `refresh_token` | Telephone -> Plugboard | Request new JWT token |
| `heartbeat` | Bidirectional | Keep connection alive |

### Environment Variable Coordination

Both components share cryptographic material and must coordinate certain values:

| Purpose | Plugboard Variable | Telephone Variable |
|---------|-------------------|-------------------|
| JWT signing | `SECRET_KEY_BASE` | `SECRET_KEY_BASE` |
| WebSocket URL | `PHX_HOST`, `PHX_PORT` | `PLUGBOARD_URL` |

**Token Flow:**
1. Admin creates telephone token in Plugboard UI/API
2. Token is provided to Telephone via `TELEPHONE_TOKEN` env var
3. Telephone connects and can refresh tokens automatically
4. Refreshed tokens are encrypted and stored in Telephone's SQLite DB

### Local Development Setup

For testing both components together:

```bash
# Terminal 1: Start Plugboard
cd Plugboard
mix phx.server
# Available at http://localhost:4000

# Terminal 2: Start a test backend
python -m http.server 8080
# Or any web server on port 8080

# Terminal 3: Start Telephone
cd Telephone
export PLUGBOARD_URL="ws://localhost:4000/telephone/websocket"
export BACKEND_HOST="localhost"
export BACKEND_PORT="8080"
export BACKEND_SCHEME="http"
# ... other required env vars
./bin/telephone
```

### Testing Across Components

1. **Unit tests** - run independently in each submodule
2. **Integration tests** - use mocks/stubs for the other component
3. **End-to-end tests** - run both components and verify full request flow

```bash
# Run Plugboard tests
cd Plugboard && mix test

# Run Telephone tests
cd Telephone && make test

# Manual E2E test
curl http://localhost:4000/call/your-path/endpoint
```

## Component-Specific Guidelines

For language and framework-specific conventions, refer to:

- **Plugboard (Elixir/Phoenix):** [Plugboard/AGENTS.md](Plugboard/AGENTS.md)
  - Phoenix 1.8 conventions
  - LiveView patterns
  - Ecto guidelines
  - Authentication handling

- **Telephone (Go):** [Telephone/AGENTS.md](Telephone/AGENTS.md)
  - Go concurrency patterns
  - Error handling
  - HTTP client guidelines
  - WebSocket best practices

## Design Principles

These principles apply across both components:

1. **Terminal Mount Points** - mount paths cannot have children, ensuring deterministic lookups
2. **CRDT-Based Clustering** - eventual consistency without coordination overhead
3. **Database as Source of Truth** - caches (ETS, Horde) are ephemeral and rebuildable
4. **Request Correlation** - UUID-based tracking for concurrent requests over a single tunnel
5. **Behavioral Testing** - tests validate observable behavior, not internal implementation
6. **Environment-Driven Config** - all configuration via environment variables, no hardcoded defaults
