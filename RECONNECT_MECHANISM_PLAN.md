# Bore Auto-Reconnect Mechanism Plan

> **Validation Status**: Validated against codebase on 2026-02-17 (commit `4851ace`).
> All line number references verified accurate. Corrections applied for:
> auth error string matching (Task 2.4/5.2), tokio minimum version for socket2
> compatibility (Task 3.1), timeout import style (Task 1.2), and existing test
> impact (Phase 5 note).

## Problem Statement

When using bore in a client-server deployment (systemd on the server, launchd on macOS client), internet disruptions cause the TCP control connection between client and server to die. The bore client process exits (gracefully or with error), and even though the service manager restarts it, several failure modes prevent automatic recovery:

1. **Silent connection death**: The client's `listen()` loop calls `recv()` with **no timeout** (`client.rs:82`). If the network drops without sending a TCP RST/FIN, `recv()` blocks indefinitely — the client hangs forever, never exits, and the service manager never restarts it.

2. **Graceful exit on EOF**: When the connection does close cleanly, `listen()` returns `Ok(())` (`client.rs:100`). The process exits with code 0. The service manager may delay restart for a "clean" exit.

3. **No retry logic**: `main.rs:73-74` calls `Client::new()` and `client.listen()` sequentially with no retry loop. Any failure is terminal.

4. **Port conflict on reconnect**: If the client reconnects before the server detects the old connection is dead, the requested port may still be bound by the old tunnel listener, causing "port already in use" errors.

5. **DNS resolution failures**: Temporary DNS outages during reconnection attempts cause immediate failure with no retry.

## Root Cause Analysis

### Current Connection Lifecycle (No Reconnection)

```
main.rs:73  Client::new()  ──► connect + auth + Hello handshake
                               │
                               ├─ Success ──► main.rs:74  client.listen()
                               │                          │
                               │                          ├─ recv() blocks indefinitely (client.rs:82)
                               │                          ├─ Heartbeat ──► ignored (no-op)
                               │                          ├─ Connection(uuid) ──► spawn proxy task
                               │                          ├─ EOF ──► return Ok(()) ──► PROCESS EXITS
                               │                          └─ Error ──► logged, continues
                               │
                               └─ Failure ──► PROCESS EXITS
```

**Key gaps:**
- `client.rs:82`: `conn.recv().await` has NO timeout — blocks forever on dead connections
- `client.rs:100`: EOF returns `Ok(())` — indistinguishable from intentional shutdown
- `main.rs:73-74`: No outer loop — any exit is final
- Server sends heartbeats every 500ms (`server.rs:147-151`) but client never validates receipt timing

### Desired Connection Lifecycle (With Reconnection)

```
main.rs  OUTER RECONNECTION LOOP:
  │
  ├─► Client::new()  ──► connect + auth + Hello handshake
  │                       │
  │                       ├─ Success ──► client.listen()
  │                       │               │
  │                       │               ├─ recv() WITH heartbeat timeout
  │                       │               ├─ Heartbeat ──► reset deadline
  │                       │               ├─ Timeout ──► return Err("heartbeat timeout")
  │                       │               ├─ EOF ──► return Err("server closed connection")
  │                       │               └─ Connection(uuid) ──► spawn proxy task
  │                       │               │
  │                       │               └─► Err propagates to outer loop
  │                       │
  │                       └─ Failure ──► classify error
  │                                      │
  │                                      ├─ Fatal (auth failure) ──► EXIT
  │                                      └─ Retriable ──► continue loop
  │
  ├─► Exponential backoff with jitter
  │
  └─► RETRY
```

---

## Implementation Plan

### Phase 1: Client Heartbeat Timeout (Foundation)

**Goal**: Make the client actively detect dead connections using the server's existing heartbeat mechanism, instead of blocking indefinitely on `recv()`.

#### [ ] Task 1.1: Add heartbeat timeout constant to `shared.rs`

**File**: `src/shared.rs`
**Change**: Add a new constant after line 21.

```rust
/// Timeout for detecting a dead control connection.
///
/// The server sends heartbeats every 500ms. If no message is received within
/// this duration, the connection is considered dead.
pub const HEARTBEAT_TIMEOUT: Duration = Duration::from_secs(8);
```

**Rationale**: 8 seconds = 16 missed heartbeats at 500ms interval. This is generous enough to handle brief network hiccups (packet loss, temporary congestion) while still detecting dead connections quickly. The value should be at least 3-4x the heartbeat interval to avoid false positives.

**Subtasks**:
- [ ] Add `HEARTBEAT_TIMEOUT` constant (8 seconds)
- [ ] Add doc comment explaining the relationship to server heartbeat interval
- [ ] Update the `use` import in `client.rs` to include `HEARTBEAT_TIMEOUT`

---

#### [ ] Task 1.2: Add timeout to `listen()` loop in `client.rs`

**File**: `src/client.rs`
**Change**: Modify the `listen()` method (lines 78-103) to wrap `recv()` in a timeout.

**Current code** (`client.rs:78-103`):
```rust
pub async fn listen(mut self) -> Result<()> {
    let mut conn = self.conn.take().unwrap();
    let this = Arc::new(self);
    loop {
        match conn.recv().await? {                    // <── NO TIMEOUT: blocks forever
            Some(ServerMessage::Heartbeat) => (),     // <── no-op, no deadline tracking
            Some(ServerMessage::Connection(id)) => { /* ... */ }
            Some(ServerMessage::Error(err)) => error!(%err, "server error"),
            None => return Ok(()),                    // <── silent exit on EOF
            // ...
        }
    }
}
```

**New code**:
```rust
pub async fn listen(mut self) -> Result<()> {
    let mut conn = self.conn.take().unwrap();
    let this = Arc::new(self);
    loop {
        match timeout(HEARTBEAT_TIMEOUT, conn.recv()).await {
            Err(_elapsed) => {
                // No message received for HEARTBEAT_TIMEOUT seconds.
                // Server sends heartbeats every 500ms, so connection is dead.
                bail!("heartbeat timeout, connection to server lost");
            }
            Ok(msg) => match msg? {
                Some(ServerMessage::Hello(_)) => warn!("unexpected hello"),
                Some(ServerMessage::Challenge(_)) => warn!("unexpected challenge"),
                Some(ServerMessage::Heartbeat) => (),
                Some(ServerMessage::Connection(id)) => {
                    let this = Arc::clone(&this);
                    tokio::spawn(
                        async move {
                            info!("new connection");
                            match this.handle_connection(id).await {
                                Ok(_) => info!("connection exited"),
                                Err(err) => warn!(%err, "connection exited with error"),
                            }
                        }
                        .instrument(info_span!("proxy", %id)),
                    );
                }
                Some(ServerMessage::Error(err)) => error!(%err, "server error"),
                None => bail!("server closed connection"),
            },
        }
    }
}
```

**Key changes**:
1. `recv()` is wrapped in `timeout(HEARTBEAT_TIMEOUT, ...)` (using the already-imported `tokio::time::timeout`)
2. Timeout expiry returns an error (retriable) instead of blocking forever
3. `None` (EOF) now returns `Err` instead of `Ok(())` — this is critical for the outer reconnection loop to trigger

**Subtasks**:
- [ ] Import `HEARTBEAT_TIMEOUT` from `shared.rs`
- [ ] Verify `timeout` is already imported at `client.rs:6` (`use tokio::{..., time::timeout}`) — no new import needed
- [ ] Wrap `conn.recv()` in timeout
- [ ] Change timeout arm to `bail!("heartbeat timeout")`
- [ ] Change `None` arm from `return Ok(())` to `bail!("server closed connection")`
- [ ] Verify existing match arms are preserved identically

---

### Phase 2: Reconnection Loop with Exponential Backoff (Core Feature)

**Goal**: Add an outer retry loop in `main.rs` that catches connection failures and reconnects with increasing delays.

#### [ ] Task 2.1: Implement exponential backoff helper

**File**: `src/shared.rs` (or inline in `main.rs` — prefer `shared.rs` for reusability)
**Change**: Add a simple exponential backoff struct. No external crate dependency — keep bore minimal.

```rust
/// Simple exponential backoff with jitter for reconnection delays.
pub struct ExponentialBackoff {
    current: Duration,
    base: Duration,
    max: Duration,
}

impl ExponentialBackoff {
    /// Create a new exponential backoff starting at `base` delay, capped at `max`.
    pub fn new(base: Duration, max: Duration) -> Self {
        Self {
            current: base,
            base,
            max,
        }
    }

    /// Get the next delay and advance the backoff state.
    /// Includes random jitter of +/- 25% to prevent thundering herd.
    pub fn next_delay(&mut self) -> Duration {
        let delay = self.current;
        self.current = (self.current * 2).min(self.max);
        // Add jitter: multiply by random factor between 0.75 and 1.25
        let jitter_factor = 0.75 + fastrand::f64() * 0.5;
        delay.mul_f64(jitter_factor)
    }

    /// Reset backoff to initial delay (call after successful connection).
    pub fn reset(&mut self) {
        self.current = self.base;
    }
}
```

**Subtasks**:
- [ ] Add `ExponentialBackoff` struct to `shared.rs`
- [ ] Implement `new()`, `next_delay()`, `reset()`
- [ ] Add jitter using existing `fastrand` dependency (already in Cargo.toml)
- [ ] Add unit test for backoff sequence and reset behavior
- [ ] Add unit test for jitter bounds (delay always between 0.75x and 1.25x expected)

---

#### [ ] Task 2.2: Add CLI flags for reconnection control

**File**: `src/main.rs`
**Change**: Add new optional flags to the `Local` subcommand.

```rust
Command::Local {
    // ... existing fields ...

    /// Disable automatic reconnection on connection loss.
    #[clap(long, default_value_t = false)]
    no_reconnect: bool,

    /// Maximum delay between reconnection attempts, in seconds.
    #[clap(long, default_value_t = 64, value_name = "SECONDS")]
    max_reconnect_delay: u64,
}
```

**Design decisions**:
- Reconnection is **ON by default** — this is the safe default for production deployments
- `--no-reconnect` restores old behavior for scripts that depend on process exit
- `--max-reconnect-delay` caps the exponential backoff (default 64 seconds)
- No `--max-reconnect-attempts` — infinite retries is the right default for a tunnel daemon. Users who want limited retries can use external tooling or the `--no-reconnect` flag with service manager restart limits.

**Subtasks**:
- [ ] Add `no_reconnect: bool` field with `#[clap(long)]`
- [ ] Add `max_reconnect_delay: u64` field with default 64
- [ ] Pass these values to the reconnection loop

---

#### [ ] Task 2.3: Implement reconnection loop in `main.rs`

**File**: `src/main.rs`
**Change**: Replace the current direct call pattern with a reconnection loop.

**Current code** (`main.rs:66-75`):
```rust
Command::Local { local_host, local_port, to, port, secret } => {
    let client = Client::new(&local_host, local_port, &to, port, secret.as_deref()).await?;
    client.listen().await?;
}
```

**New code**:
```rust
Command::Local {
    local_host, local_port, to, port, secret,
    no_reconnect, max_reconnect_delay,
} => {
    // First attempt — propagate errors directly for immediate feedback
    let client = Client::new(&local_host, local_port, &to, port, secret.as_deref()).await?;

    if no_reconnect {
        // Legacy behavior: exit on any disconnection
        client.listen().await?;
    } else {
        // Reconnection mode: retry on transient failures
        let mut backoff = ExponentialBackoff::new(
            Duration::from_secs(1),
            Duration::from_secs(max_reconnect_delay),
        );

        // Run the first listen (we already have a connected client)
        match client.listen().await {
            Ok(()) => unreachable!("listen() now always returns Err"),
            Err(e) => warn!("connection lost: {e:#}"),
        }

        // Reconnection loop
        loop {
            let delay = backoff.next_delay();
            info!("reconnecting in {delay:.1?}...");
            tokio::time::sleep(delay).await;

            match Client::new(
                &local_host, local_port, &to, port, secret.as_deref()
            ).await {
                Ok(client) => {
                    backoff.reset();
                    info!("reconnected successfully");
                    match client.listen().await {
                        Ok(()) => unreachable!("listen() now always returns Err"),
                        Err(e) => {
                            if is_auth_error(&e) {
                                return Err(e);
                            }
                            warn!("connection lost: {e:#}");
                        }
                    }
                }
                Err(e) => {
                    if is_auth_error(&e) {
                        return Err(e);
                    }
                    warn!("reconnection failed: {e:#}");
                }
            }
        }
    }
}
```

**Key design choices**:
1. **First connection attempt propagates errors directly** — if the server address is wrong, auth fails on first try, etc., the user gets immediate feedback instead of watching infinite retries.
2. **Subsequent failures retry** — after the first successful connection, we know the config is valid, so failures are transient.
3. **Auth errors are always fatal** — wrong secret won't fix itself, so we don't retry. This is detected by checking the error message.
4. **Backoff resets on successful connection** — if the tunnel runs for hours then drops, we start with short delays again.

**Subtasks**:
- [ ] Destructure new CLI fields in the match arm
- [ ] Create `ExponentialBackoff` with configured max delay
- [ ] Keep first `Client::new()` call outside the loop (fail fast on first attempt)
- [ ] Add reconnection loop after first disconnection
- [ ] Implement `is_auth_error()` helper function
- [ ] Reset backoff on successful `Client::new()`
- [ ] Add info/warn logging for reconnection state transitions
- [ ] Import `Duration` and `ExponentialBackoff` in `main.rs`

---

#### [ ] Task 2.4: Implement error classification

**File**: `src/main.rs` (or `src/shared.rs`)
**Change**: Add a helper to distinguish fatal errors from retriable ones.

```rust
/// Check if an error is an authentication error that should not be retried.
fn is_auth_error(err: &anyhow::Error) -> bool {
    let msg = format!("{err:#}");
    msg.contains("server requires authentication")
        || msg.contains("invalid secret")
        || msg.contains("server requires secret")
        || msg.contains("expected authentication challenge")
}
```

**Matched error paths** (verified against source):
- `"server requires authentication, but no client secret was provided"` — `client.rs:54`, when server sends `Challenge` but client has no secret
- `"server error: invalid secret"` — `auth.rs:59` → `server.rs:123` → `client.rs:52`, when HMAC validation fails
- `"server error: server requires secret, but no secret was provided"` — `auth.rs:62` → `server.rs:123` → `client.rs:52`, when client skips auth but server requires it
- `"expected authentication challenge, but no secret was required"` — `auth.rs:73`, when client has secret but server doesn't require one

**Note**: String matching on error messages is fragile but pragmatic here. The alternative (custom error types throughout the codebase) would require significant refactoring. The error messages being matched are all hardcoded strings in the bore source code, so they're stable.

**Subtasks**:
- [ ] Implement `is_auth_error()` function
- [ ] Verify all auth-related error messages in `auth.rs` and `client.rs` are covered (see matched paths above)
- [ ] Add unit test with sample error messages

---

### Phase 3: TCP Keepalive (Defense in Depth)

**Goal**: Configure OS-level TCP keepalive on control connections to detect dead connections even if the application-level heartbeat mechanism fails.

#### [ ] Task 3.1: Add `socket2` dependency

**File**: `Cargo.toml`
**Change**: Add `socket2` crate for TCP keepalive configuration.

```toml
[dependencies]
socket2 = { version = "0.5", features = ["all"] }
```

**Rationale**: `socket2` provides cross-platform TCP keepalive configuration that `tokio::net::TcpStream` doesn't expose directly. It's a well-maintained, zero-overhead wrapper around OS socket APIs.

**IMPORTANT — Tokio version bump required**: `socket2::SockRef::from()` requires the `AsFd` trait (on Unix). Tokio's `TcpStream` only implements `AsFd` since **tokio 1.21.0** (Aug 2022). The current `Cargo.toml` specifies minimum `tokio >= 1.17.0`. This must be bumped:

```toml
tokio = { version = "1.21.0", features = ["rt-multi-thread", "io-util", "macros", "net", "time"] }
```

The actual resolved version is already 1.28.0 (via Cargo.lock), so no downstream impact. This change only updates the declared minimum to match the actual API requirements.

**Subtasks**:
- [ ] Add `socket2` to `[dependencies]` in `Cargo.toml`
- [ ] Bump minimum `tokio` version from `1.17.0` to `1.21.0` in both `[dependencies]` and `[dev-dependencies]`
- [ ] Verify it compiles on Linux (server) and macOS (client)

---

#### [ ] Task 3.2: Create TCP keepalive configuration helper

**File**: `src/shared.rs`
**Change**: Add a function to configure TCP keepalive on a `TcpStream`.

```rust
use socket2::{SockRef, TcpKeepalive};
use tokio::net::TcpStream;

/// Configure TCP keepalive on a stream for faster dead connection detection.
///
/// This sets the OS to start probing after 30s of idle, probe every 10s,
/// and give up after 3 failed probes (~60s total to detect a dead connection).
///
/// Requires tokio >= 1.21.0 for `AsFd` impl on `TcpStream`.
pub fn set_tcp_keepalive(stream: &TcpStream) -> Result<()> {
    let sock_ref = SockRef::from(stream);
    let keepalive = TcpKeepalive::new()
        .with_time(Duration::from_secs(30))
        .with_interval(Duration::from_secs(10))
        .with_retries(3);  // Supported on Linux and macOS; not on Windows
    sock_ref
        .set_tcp_keepalive(&keepalive)
        .context("failed to set TCP keepalive")?;
    Ok(())
}
```

**Platform support for `.with_retries()`**: Supported on Linux (via `TCP_KEEPCNT`) and macOS — both target platforms per the problem statement. Not supported on Windows or some niche platforms (OpenBSD, Haiku, Redox). Since bore's reconnect use case targets Linux servers and macOS clients, this is fine as-is. If Windows support is ever needed, use `#[cfg]` to conditionally omit `.with_retries()`.

**Note on `SockRef::from(stream)`**: This calls `AsFd::as_fd()` on the tokio `TcpStream`. The reference should be passed directly (not `&stream`) since `SockRef::from` takes `&impl AsFd`.

**Subtasks**:
- [ ] Implement `set_tcp_keepalive()` function
- [ ] Verify `.with_retries()` compiles on Linux and macOS (target platforms)
- [ ] Add doc comment explaining the timing parameters and `AsFd` requirement

---

#### [ ] Task 3.3: Apply TCP keepalive to client control connection

**File**: `src/client.rs`
**Change**: In `Client::new()`, after establishing the control connection, set TCP keepalive.

After line 43 (`let mut stream = Delimited::new(connect_with_timeout(to, CONTROL_PORT).await?);`):
```rust
// Set TCP keepalive for faster dead connection detection
set_tcp_keepalive(stream.get_ref())?;
```

**Note**: `Delimited<U>` doesn't currently expose `get_ref()`. A small addition to `shared.rs` is needed:
```rust
impl<U> Delimited<U> {
    /// Get a reference to the underlying transport stream.
    pub fn get_ref(&self) -> &U {
        self.0.get_ref()
    }
}
```

**Subtasks**:
- [ ] Add `get_ref()` method to `Delimited<U>` in `shared.rs`
- [ ] Call `set_tcp_keepalive()` on the control stream in `Client::new()`
- [ ] Also apply to per-connection streams in `handle_connection()` (optional, lower priority)

---

#### [ ] Task 3.4: Apply TCP keepalive to server control connections

**File**: `src/server.rs`
**Change**: In `handle_connection()`, after accepting the stream, set TCP keepalive.

At line 118 (`async fn handle_connection(&self, stream: TcpStream)`), before wrapping in `Delimited`:
```rust
set_tcp_keepalive(&stream)?;
let mut stream = Delimited::new(stream);
```

**Subtasks**:
- [ ] Apply `set_tcp_keepalive()` to incoming control connections
- [ ] Import the function from `shared.rs`

---

### Phase 4: Server-Side Robustness (Edge Cases)

**Goal**: Ensure the server handles rapid client reconnection gracefully.

#### [ ] Task 4.1: Handle port-in-use during client reconnection

**Context**: When a client disconnects and reconnects quickly (requesting the same port), the old server task may still be running because:
- The heartbeat send at `server.rs:147` hasn't failed yet (TCP write buffers haven't flushed)
- TCP keepalive hasn't triggered yet
- The old `TcpListener` still holds the port

**Solution**: This is already handled by the reconnection loop's exponential backoff. The server returns `ServerMessage::Error("port already in use")`, the client receives this error in `Client::new()`, classifies it as retriable, and retries after a delay. By the next attempt, the old server task will have detected the dead connection and released the port.

**No code change needed** — the existing behavior + reconnection loop handles this. But we should verify and document this.

**Subtasks**:
- [ ] Verify that "port already in use" error from server doesn't match `is_auth_error()`
- [ ] Add integration test for rapid reconnection with same port
- [ ] Document this behavior in code comments

---

#### [ ] Task 4.2: Add server-side heartbeat response requirement (Optional / Future Enhancement)

**Current behavior**: Server sends `Heartbeat`, client ignores it. Server detects dead client only when `stream.send(Heartbeat)` fails — which depends on TCP write buffer flushing.

**Enhanced behavior** (optional, for future consideration): Server expects a `HeartbeatAck` response from client. If no ack within N seconds, server proactively closes the connection and frees the port.

**This is NOT included in the initial implementation** because:
1. It requires a protocol change (new `ClientMessage::HeartbeatAck` variant)
2. TCP keepalive (Phase 3) already provides faster dead-connection detection
3. The current approach (client-side heartbeat timeout + reconnection) is sufficient

**Subtasks**:
- [ ] Document as future enhancement
- [ ] No implementation in this phase

---

### Phase 5: Testing (Verification)

**Goal**: Comprehensive test coverage for all reconnection scenarios.

**NOTE — Existing test impact**: The `spawn_client()` helper in `tests/e2e_test.rs:30` spawns `client.listen()` via `tokio::spawn(client.listen())`. After Phase 1 changes, `listen()` always returns `Err` (never `Ok(())`). Since the spawned task's result is dropped (not `.await`ed), existing tests still pass — the error is silently ignored. However, for clarity and to avoid confusing error logs during test runs, consider adding a `.map_err(|e| warn!(...))` or similar handling in `spawn_client()`.

#### [ ] Task 5.1: Unit test for `ExponentialBackoff`

**File**: `src/shared.rs` (inline `#[cfg(test)]` module) or `tests/backoff_test.rs`

```rust
#[test]
fn test_backoff_sequence() {
    let mut backoff = ExponentialBackoff::new(
        Duration::from_secs(1),
        Duration::from_secs(30),
    );
    // Delays should roughly double: 1, 2, 4, 8, 16, 30 (capped), 30, ...
    // With jitter, each delay is between 0.75x and 1.25x the base
    for expected_base in [1, 2, 4, 8, 16, 30, 30] {
        let delay = backoff.next_delay();
        let min = Duration::from_secs(expected_base).mul_f64(0.75);
        let max = Duration::from_secs(expected_base).mul_f64(1.25);
        assert!(delay >= min && delay <= max, "delay {delay:?} out of range [{min:?}, {max:?}]");
    }
}

#[test]
fn test_backoff_reset() {
    let mut backoff = ExponentialBackoff::new(
        Duration::from_secs(1),
        Duration::from_secs(60),
    );
    backoff.next_delay(); // 1s
    backoff.next_delay(); // 2s
    backoff.next_delay(); // 4s
    backoff.reset();
    let delay = backoff.next_delay();
    // After reset, should be back to ~1s (with jitter)
    assert!(delay < Duration::from_secs(2));
}
```

**Subtasks**:
- [ ] Test exponential growth sequence
- [ ] Test max cap is respected
- [ ] Test reset returns to base delay
- [ ] Test jitter bounds

---

#### [ ] Task 5.2: Unit test for error classification

**File**: `tests/reconnect_test.rs` or inline in `main.rs`

```rust
#[test]
fn test_auth_error_detection() {
    // Fatal auth errors — should NOT be retried
    assert!(is_auth_error(&anyhow::anyhow!("server requires authentication, but no client secret was provided")));
    assert!(is_auth_error(&anyhow::anyhow!("server error: invalid secret")));
    assert!(is_auth_error(&anyhow::anyhow!("server error: server requires secret, but no secret was provided")));
    assert!(is_auth_error(&anyhow::anyhow!("expected authentication challenge, but no secret was required")));

    // Retriable errors — should be retried
    assert!(!is_auth_error(&anyhow::anyhow!("could not connect to server:7835")));
    assert!(!is_auth_error(&anyhow::anyhow!("heartbeat timeout, connection to server lost")));
    assert!(!is_auth_error(&anyhow::anyhow!("server error: port already in use")));
    assert!(!is_auth_error(&anyhow::anyhow!("server closed connection")));
}
```

**Subtasks**:
- [ ] Test auth errors are classified as fatal
- [ ] Test connection errors are classified as retriable
- [ ] Test heartbeat timeout is classified as retriable
- [ ] Test port conflict is classified as retriable

---

#### [ ] Task 5.3: Integration test — reconnection after server restart

**File**: `tests/e2e_test.rs`

```rust
#[tokio::test]
async fn reconnect_after_server_restart() {
    // 1. Start server
    // 2. Start client with reconnection enabled
    // 3. Verify tunnel works (send/receive data)
    // 4. Kill server (drop the server handle)
    // 5. Wait for client to detect disconnection (~8s heartbeat timeout)
    // 6. Restart server
    // 7. Wait for client to reconnect (up to backoff delay)
    // 8. Verify tunnel works again
}
```

**Subtasks**:
- [ ] Set up test infrastructure for server restart
- [ ] Verify client detects disconnection via heartbeat timeout
- [ ] Verify client reconnects after server restart
- [ ] Verify tunnel is functional after reconnection
- [ ] Use `SERIAL_GUARD` mutex for test isolation

---

#### [ ] Task 5.4: Integration test — auth failure does not retry

**File**: `tests/e2e_test.rs`

```rust
#[tokio::test]
async fn auth_failure_no_retry() {
    // 1. Start server with secret "correct_secret"
    // 2. Start client with secret "wrong_secret" and reconnection enabled
    // 3. Verify client exits with auth error (not infinite retry)
}
```

**Subtasks**:
- [ ] Start server with one secret, client with another
- [ ] Verify client exits immediately (doesn't retry)
- [ ] Verify error message indicates auth failure

---

#### [ ] Task 5.5: Integration test — `--no-reconnect` flag

**File**: `tests/e2e_test.rs`

```rust
#[tokio::test]
async fn no_reconnect_flag_exits_on_disconnect() {
    // 1. Start server
    // 2. Start client with --no-reconnect
    // 3. Kill server
    // 4. Verify client exits (does not retry)
}
```

**Subtasks**:
- [ ] Test that `--no-reconnect` preserves legacy exit behavior
- [ ] Verify process exits after connection loss

---

#### [ ] Task 5.6: Integration test — rapid reconnection with same port

**File**: `tests/e2e_test.rs`

```rust
#[tokio::test]
async fn reconnect_same_port() {
    // 1. Start server
    // 2. Start client requesting port 12345
    // 3. Verify tunnel on port 12345 works
    // 4. Kill and restart server
    // 5. Verify client reconnects and gets port 12345 again
}
```

**Subtasks**:
- [ ] Test specific port re-binding after reconnection
- [ ] Verify the client gets the same port after reconnection
- [ ] Handle transient "port in use" errors during the test

---

### Phase 6: Documentation (Polish)

#### [ ] Task 6.1: Update README with reconnection behavior

**File**: `README.md`

Add a section documenting:
- Automatic reconnection is enabled by default
- Exponential backoff behavior (1s → 2s → 4s → ... → 64s max)
- How to disable with `--no-reconnect`
- How to tune with `--max-reconnect-delay`
- Auth failures are never retried

**Subtasks**:
- [ ] Add "Reconnection" section to README
- [ ] Document CLI flags
- [ ] Add example showing reconnection in logs

---

#### [ ] Task 6.2: Update CLAUDE.md architecture section

**File**: `CLAUDE.md`

Update the architecture documentation to reflect:
- New heartbeat timeout behavior
- Reconnection loop in `main.rs`
- `ExponentialBackoff` in `shared.rs`
- TCP keepalive configuration
- New CLI flags

**Subtasks**:
- [ ] Update Architecture section
- [ ] Update Key Patterns section
- [ ] Update Protocol Flow section

---

## File Change Summary

| File | Changes | Lines (est.) |
|------|---------|--------------|
| `src/shared.rs` | Add `HEARTBEAT_TIMEOUT`, `ExponentialBackoff`, `set_tcp_keepalive()`, `get_ref()` | +60 |
| `src/client.rs` | Heartbeat timeout in `listen()`, TCP keepalive in `new()` | +15, -5 |
| `src/main.rs` | CLI flags, reconnection loop, `is_auth_error()` | +60, -3 |
| `src/server.rs` | TCP keepalive on control connections | +3 |
| `Cargo.toml` | Add `socket2` dependency, bump tokio minimum to 1.21.0 | +1, ~2 |
| `tests/e2e_test.rs` | Reconnection integration tests | +120 |
| `README.md` | Documentation update | +30 |
| `CLAUDE.md` | Architecture update | +15 |
| **Total** | | **~300 lines added** |

## Dependency Changes

| Crate | Version | Purpose | Size Impact |
|-------|---------|---------|-------------|
| `socket2` | 0.5 | TCP keepalive configuration | Minimal (thin OS wrapper) |
| `tokio` | 1.17.0 → 1.21.0 (minimum bump) | Required for `AsFd` impl on `TcpStream` (used by `socket2`) | No change (resolved version already 1.28.0) |

No other new dependencies. The exponential backoff is implemented inline using the existing `fastrand` crate (v1.9.0, which provides the `f64()` function used for jitter).

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Heartbeat timeout too aggressive (false positives) | Low | Client unnecessarily reconnects | 8s is 16x the heartbeat interval; tunable via constant |
| Port conflict during rapid reconnect | Medium | Temporary "port in use" errors | Exponential backoff naturally handles this; server cleans up within seconds |
| Backoff too slow for production use | Low | Longer downtime during recovery | 1s initial delay is fast; configurable via `--max-reconnect-delay` |
| String-based error classification misses edge cases | Low | Auth error retried unnecessarily | Error messages are hardcoded in source; add tests |
| `socket2` platform incompatibility | Very Low | TCP keepalive not set | Feature is defense-in-depth; app-level heartbeat is primary mechanism |
| Breaking change for scripts depending on exit behavior | Medium | Scripts break | `--no-reconnect` flag preserves old behavior |

## Implementation Order and Dependencies

```
Phase 1 ────────────────────────────────────────────────►
  Task 1.1 (constant) ──► Task 1.2 (heartbeat timeout)
                                       │
Phase 2 ───────────────────────────────┼────────────────►
  Task 2.1 (backoff) ─┐               │
  Task 2.2 (CLI) ─────┼──► Task 2.3 (reconnect loop)
  Task 2.4 (classify) ─┘
                                       │
Phase 3 ───────────────────────────────┼────────────────►
  Task 3.1 (socket2) ──► Task 3.2 ──► Task 3.3 + 3.4
                                       │
Phase 4 ───────────────────────────────┼────────────────►
  Task 4.1 (verify) ──────────────────►│
                                       │
Phase 5 ───────────────────────────────┼────────────────►
  Task 5.1-5.6 (all tests) ───────────►│
                                       │
Phase 6 ───────────────────────────────┼────────────────►
  Task 6.1-6.2 (docs) ────────────────►│
```

- **Phase 1 must complete before Phase 2** (reconnection loop depends on `listen()` returning errors)
- **Phase 2 can start Task 2.1, 2.2, 2.4 in parallel** (they converge at Task 2.3)
- **Phase 3 is independent** of Phases 1-2 (can be done in parallel)
- **Phase 4 is verification only** (no code changes, just testing)
- **Phase 5 depends on Phases 1-3** (tests cover all new functionality)
- **Phase 6 depends on Phase 5** (document verified behavior)

## Acceptance Criteria

1. **Core functionality**: Client automatically reconnects to server after internet disruption without manual intervention
2. **Detection speed**: Dead connections detected within 8 seconds (heartbeat timeout)
3. **Backoff behavior**: Reconnection delays follow exponential pattern (1s, 2s, 4s, 8s, 16s, 32s, 64s max)
4. **Auth safety**: Authentication failures cause immediate exit, not infinite retry
5. **Backward compatibility**: `--no-reconnect` flag preserves old exit-on-disconnect behavior
6. **Port stability**: Client re-obtains the same port after reconnection (when using `--port`)
7. **Logging**: All reconnection events are clearly logged with timestamps and delays
8. **Tests pass**: All existing tests pass, new reconnection tests pass
9. **No new unsafe code**: Maintains `#![forbid(unsafe_code)]`
10. **Minimal dependencies**: Only `socket2` added (thin OS wrapper)
