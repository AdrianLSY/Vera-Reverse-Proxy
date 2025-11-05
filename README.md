# Vera-Stack

**A dynamic, database-driven reverse proxy system with WebSocket tunneling**

[![Elixir](https://img.shields.io/badge/elixir-1.17-purple.svg)](https://elixir-lang.org)
[![Phoenix](https://img.shields.io/badge/phoenix-1.7-orange.svg)](https://phoenixframework.org)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

---

## Overview

Vera-Stack is a complete reverse proxy solution that replaces static configuration files with **dynamic, application-driven routing** via persistent WebSocket connections. Applications register themselves through lightweight **"telephone" sidecars**, which establish secure tunnels to the **Plugboard** reverse proxy. This eliminates the need for config files, restarts, or manual port forwarding.

### System Components

* **[Plugboard](Plugboard/)** - The reverse proxy server that handles incoming requests and routes them through WebSocket tunnels
* **[Telephone](Telephone/)** - The lightweight sidecar that runs alongside your applications and maintains tunnels to Plugboard

### Highlights

* **Zero-configuration routing** - backends register dynamically at runtime
* **NAT/firewall traversal** - route to hosts behind private networks
* **Database-driven** - all routes managed through UI or API
* **Cluster-ready** - distributed consensus via CRDTs for resilience
* **Auto-scaling aware** - services register/deregister automatically

### Typical Use Cases

* Exposing local servers in development
* Dynamic tenant routing in multi-tenant SaaS
* Auto-scaling environments in the cloud
* Edge or IoT devices behind NAT
* Centralized routing for microservice environments

---

## Architecture

### System Overview

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

---

## How It Works

### Request Lifecycle

1. Client sends a request to Plugboard (e.g., `GET /api/users`).
2. `ProxyController` performs an O(1) path lookup in ETS.
3. Distributed registry locates the corresponding telephone process.
4. The request is sent over the WebSocket via `TelephoneChannel`.
5. The telephone forwards it to the local backend (e.g., `localhost:3000`).
6. The backend processes the request and responds.
7. The telephone returns the response over the same WebSocket.
8. Plugboard streams the response back to the original client.

### Design Principles

1. **Terminal Mount Points** - mount paths cannot have children, ensuring deterministic lookups.
2. **CRDT-Based Clustering** - eventual consistency without coordination overhead.
3. **Database as Source of Truth** - ETS and Horde caches are ephemeral and rebuildable.
4. **Request Correlation** - UUID-based tracking for concurrent requests over a single tunnel.
5. **Soft Deletes** - `deleted_at` timestamps preserve historical path data.
6. **Behavioral Testing** - tests validate observable behavior, not internal implementation.

---

## Getting Started

### Quick Start

```bash
# Clone the repository
git clone https://github.com/yourusername/vera-stack.git
cd vera-stack

# Start Plugboard (see Plugboard/README.md for detailed setup)
cd Plugboard
mix deps.get
mix ecto.setup
mix phx.server

# Plugboard will be available at http://localhost:4000
```

For detailed setup, configuration, and deployment instructions, see:
* **[Plugboard/README.md](Plugboard/README.md)** - Reverse proxy setup and configuration
* **[Telephone/README.md](Telephone/README.md)** - Sidecar deployment (coming soon)

---

## Documentation

### Component Documentation

* **[Plugboard/README.md](Plugboard/README.md)** - Reverse proxy server
  * Setup and configuration
  * Internal architecture
  * API reference
  * Testing guidelines
  * Development conventions

* **[Telephone/README.md](Telephone/README.md)** - Sidecar component
  * Installation and deployment
  * Configuration options
  * Integration patterns

### Additional Resources

* **[Plugboard/AGENTS.md](Plugboard/AGENTS.md)** - Development conventions and Phoenix guidelines
* **[Plugboard/TESTING_GUIDELINES.md](Plugboard/TESTING_GUIDELINES.md)** - Testing strategy and patterns
* **[Plugboard/FUTURE_WORK.md](Plugboard/FUTURE_WORK.md)** - Roadmap and planned features
* **[Plugboard/CONTRIBUTING.md](Plugboard/CONTRIBUTING.md)** - Contribution workflow and standards

---

## Roadmap

* **Load Balancing** - distribute traffic across multiple backend instances
* **Observability** - distributed tracing, metrics, and logging
* **Multi-domain Routing** - virtual hosts and wildcard domain support
* **Rate Limiting** - per-path and per-client rate limiting
* **SSL/TLS Termination** - automatic certificate management
* **Telephone Implementation** - Complete the sidecar component

See [Plugboard/FUTURE_WORK.md](Plugboard/FUTURE_WORK.md) for detailed roadmap.

---

## Technology Stack

* **Phoenix Framework 1.7** - web and WebSocket layer
* **Elixir 1.17** - concurrent and fault-tolerant runtime
* **PostgreSQL 14+** - canonical data store for routes
* **ETS** - in-memory cache for O(1) path lookup
* **Horde** - distributed registry with CRDT-based consistency
* **libcluster** - automatic cluster formation and recovery
* **Phoenix Channels** - bidirectional WebSocket communication
* **Joken** - JWT authentication for sidecar telephones

---

## License

Licensed under the **MIT License**. See [LICENSE](LICENSE) for details.

---

## Acknowledgments

### Built With

* [Phoenix Framework](https://phoenixframework.org) - by Chris McCord and contributors
* [Horde](https://github.com/derekkraan/horde) - distributed registry by Derek Kraan
* [libcluster](https://github.com/bitwalker/libcluster) - clustering by Paul Schoenfelder

### Inspired By

* [HAProxy](https://www.haproxy.org/) - High-performance load balancing and proxying
* [Nginx](https://nginx.org/) - Reverse proxy and web server architecture
* [Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/) - Zero-trust networking and secure tunneling

### Supported By

* [Rooftop Energy](https://rooftop.my/) - Thanks to the team for allowing me to build out my crazy idea!

---

**Made with Love, Robots, Elixir & Go**
