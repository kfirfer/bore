# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bore is a modern, simple TCP tunnel in Rust that exposes local ports to a remote server, bypassing NAT. It's an alternative to localtunnel/ngrok. The crate is published as `bore-cli` with binary name `bore`.

## Build & Development Commands

```bash
cargo build --all-features        # Build
cargo test                         # Run all tests
cargo test basic_proxy             # Run a single test by name
cargo fmt -- --check               # Check formatting
cargo clippy -- -D warnings        # Lint (CI treats warnings as errors)
RUST_LOG=debug cargo test          # Run tests with debug logging
```

## Architecture

The codebase is ~400 lines of async Rust using Tokio. No unsafe code (`#![forbid(unsafe_code)]`).

### Modules

- **`main.rs`** — CLI entry point using clap. Two subcommands: `local` (client) and `server`.
- **`shared.rs`** — Protocol definitions. `ClientMessage`/`ServerMessage` enums serialized as JSON over TCP with null-byte delimiters. `Delimited<U>` wraps any async stream for framed JSON I/O. Key constants: `CONTROL_PORT = 7835`, `MAX_FRAME_LENGTH = 256`, `NETWORK_TIMEOUT = 3s`.
- **`auth.rs`** — Optional HMAC-SHA256 challenge-response authentication. Secret is SHA256-hashed before use. Constant-time comparison.
- **`client.rs`** — `Client` connects to server's control port, sends `Hello(port)`, receives assigned port. For each incoming `Connection(uuid)`, opens a new TCP connection, sends `Accept(uuid)`, then bidirectionally proxies between local service and tunnel.
- **`server.rs`** — `Server` listens on control port. Allocates tunnel ports (random selection, 150 attempts). Stores pending connections in `DashMap<Uuid, TcpStream>` with 10-second expiry. Sends heartbeats every 500ms.

### Protocol Flow

1. Client connects to server on control port (7835)
2. Optional auth: server sends `Challenge(uuid)`, client responds with `Authenticate(hmac)`
3. Client sends `Hello(desired_port)`, server responds with `Hello(actual_port)` and starts tunnel listener
4. When external traffic hits the tunnel port, server stores the connection by UUID, sends `Connection(uuid)` to client
5. Client opens a new connection to server, sends `Accept(uuid)`, server pairs streams, bidirectional copy begins

### Key Patterns

- `Delimited<U>` for framed JSON messaging with null-byte delimiters (via `tokio_util::codec`)
- `Arc<DashMap>` for lock-free concurrent pending connection storage
- `Arc<Client>`/`Arc<Server>` shared across spawned Tokio tasks
- `tokio::io::copy_bidirectional` for efficient TCP proxying
- `anyhow::Result` with `.context()` for error propagation

## Testing

Tests are in `tests/e2e_test.rs` (integration) and `tests/auth_test.rs` (auth unit tests). Integration tests use a `lazy_static` mutex (`SERIAL_GUARD`) to run serially and avoid port conflicts. CI retries tests up to 3 times due to potential port contention.
