# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Important: Read AGENTS.md First

**Always read `AGENTS.md` before making changes.** It contains monorepo conventions, submodule workflows, and integration patterns for coordinating work across Plugboard and Telephone.

## Project Overview

Vera-Stack is a monorepo containing a complete dynamic reverse proxy system with WebSocket tunneling. It replaces static configuration files with application-driven routing via persistent WebSocket connections.

### Components

| Component | Language | Description |
|-----------|----------|-------------|
| **[Plugboard](Plugboard/)** | Elixir/Phoenix | Reverse proxy server handling incoming requests and routing through WebSocket tunnels |
| **[Telephone](Telephone/)** | Go | Lightweight sidecar maintaining tunnels to Plugboard and forwarding requests to backends |

## Repository Structure

```
vera-stack/
├── CLAUDE.md              # This file - monorepo guidance
├── AGENTS.md              # Development conventions
├── README.md              # Project overview
├── LICENSE
├── .github/
│   └── workflows/         # CI/CD automation
│       └── update-submodule.yml
├── Plugboard/             # Git submodule - Phoenix reverse proxy
│   ├── CLAUDE.md          # Plugboard-specific guidance
│   ├── AGENTS.md          # Phoenix/Elixir conventions
│   └── ...
└── Telephone/             # Git submodule - Go sidecar
    ├── CLAUDE.md          # Telephone-specific guidance
    ├── AGENTS.md          # Go conventions
    └── ...
```

## Build & Development Commands

```bash
# Clone with submodules
git clone --recurse-submodules <repo-url>

# Or initialize submodules after clone
git submodule init
git submodule update

# Update submodules to latest commits
git submodule update --remote

# Update a specific submodule
git submodule update --remote Plugboard
git submodule update --remote Telephone

# Check submodule status
git submodule status
```

### Working in Submodules

When making changes to a component, work directly in the submodule directory:

```bash
# Plugboard development
cd Plugboard
mix deps.get
mix ecto.setup
mix phx.server
mix precommit              # Run before committing

# Telephone development
cd Telephone
make build
make test
make precommit             # Run before committing
```

**Important:** Commits in submodules are separate from the parent repo. After committing and pushing in a submodule, the parent repo's reference is automatically updated via CI/CD.

## System Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ HTTP Request: GET /api/users
       ▼
┌───────────────────────────────────┐
│         Plugboard Cluster         │
│  ┌────────────────────────────┐   │
│  │   ProxyController          │   │
│  │   1. Match /api in ETS     │   │
│  │   2. Find telephone in     │   │
│  │      distributed registry  │   │
│  └────────────────────────────┘   │
│               │                   │
│               │ WebSocket Tunnel  │
│               ▼                   │
│  ┌────────────────────────────┐   │
│  │   TelephoneChannel         │   │
│  │   (WebSocket connection)   │   │
│  └────────────────────────────┘   │
└────────┬──────────────────────────┘
         │ Proxy Request over WebSocket
         ▼
┌──────────────────────────────┐
│   Container / Pod            │
│  ┌────────────────────────┐  │
│  │  Telephone (Sidecar)   │  │
│  │  - Maintains WebSocket │  │
│  │  - Intercepts traffic  │  │
│  └────────┬───────────────┘  │
│           │ localhost:PORT   │
│           ▼                  │
│  ┌────────────────────────┐  │
│  │ Your Web Server        │  │
│  │ (Rails, Express,       │  │
│  │  Django, Spring, etc.) │  │
│  └────────────────────────┘  │
└──────────────────────────────┘
```

### Request Flow

1. Client sends request to Plugboard (e.g., `GET /api/users`)
2. `ProxyController` performs O(1) path lookup in ETS
3. Distributed registry locates the corresponding telephone process
4. Request sent over WebSocket via `TelephoneChannel`
5. Telephone forwards to local backend (e.g., `localhost:3000`)
6. Backend processes request and responds
7. Telephone returns response over the same WebSocket
8. Plugboard streams response back to client

## Component Documentation

For component-specific details, refer to:

| Component | CLAUDE.md | AGENTS.md |
|-----------|-----------|-----------|
| Plugboard | [Plugboard/CLAUDE.md](Plugboard/CLAUDE.md) | [Plugboard/AGENTS.md](Plugboard/AGENTS.md) |
| Telephone | [Telephone/CLAUDE.md](Telephone/CLAUDE.md) | [Telephone/AGENTS.md](Telephone/AGENTS.md) |

## CI/CD & Automation

### Automatic Submodule Sync

The repository uses GitHub Actions to automatically sync submodule references:

1. When changes are pushed to a submodule's main branch, it triggers a `repository_dispatch` event
2. The `update-submodule.yml` workflow receives the event
3. It updates the parent repo's submodule reference to the latest commit
4. A commit is automatically created: `chore: update <submodule> to <short-sha>`

**Workflow location:** `.github/workflows/update-submodule.yml`

**Required secrets:**
- `VERA_AUTOMATION_APP_ID` - GitHub App ID for authentication
- `VERA_AUTOMATION_PRIVATE_KEY` - GitHub App private key

### Manual Submodule Update

If automatic sync fails or you need to update manually:

```bash
git submodule update --remote Plugboard
git add Plugboard
git commit -m "chore: update Plugboard to latest"
git push
```
