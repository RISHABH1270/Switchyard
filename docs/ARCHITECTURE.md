# Architecture

> **Scope of this document:** Rungs 1-3 only.
>
> Rungs 4-12 are intentionally *not* designed here. Systems-level design decisions made months in advance almost always turn out wrong once real code hits the kernel. We'll write a fresh design section for each rung as we approach it, informed by what we've learned building the previous rung.

---

## Guiding Principles

These shape every design decision below.

1. **State is a first-class primitive.** Even at Rung 1, we design the connection lifecycle in a way that will let hot state swaps land cleanly at Rung 10. This mainly means: no global mutable state, ever. Every piece of shared state lives behind an interface with a clear ownership model.
2. **The hot path must be observable.** Every request will eventually emit a structured trace. We start noting phase boundaries (accept → parse → route → forward → response) from Rung 1, even if we only `printf` them today.
3. **Fail loudly in development, gracefully in production.** Assertion-heavy debug builds. Explicit error propagation, not exceptions on the hot path.
4. **Zero-copy where cheap, not where clever.** We won't chase `sendfile`/`splice` yet. But we won't add gratuitous copies either.
5. **Testable from Rung 1.** Every rung ships with tests. No rung is "done" until its tests pass under a `make test` target.

---

## Target Environment

- **OS:** Linux (primary) and macOS (dev). Windows is out of scope.
- **Compiler:** GCC 13+ or Clang 16+ with C++20.
- **Build:** CMake 3.20+.
- **Test framework:** GoogleTest (unit) + a small integration harness written in Python (spawn proxy → curl → assert output).
- **Dependencies for Rungs 1-3:** none. Pure standard library + POSIX sockets. We deliberately avoid pulling in Boost/ASIO/libevent early — the point is to understand what those libraries hide.

---

## Rung 1 — TCP Echo Server

### Goal

Accept a TCP connection on a configurable port. Read bytes from the client. Write the same bytes back. Close cleanly. Handle multiple clients concurrently.

**Not a proxy yet.** This rung exists to make sockets, event loops, and connection lifecycle concrete before we add HTTP on top.

### Design

**I/O model:** single-threaded event loop using `epoll` on Linux, `kqueue` on macOS, abstracted behind a small `Reactor` class. No thread pool yet — we want to feel where blocking hurts before we solve it.

**Why not threads-per-connection?** It works for 10 clients and falls over at 10,000. Real proxies use event-driven I/O. Learning that model now pays off for every rung after.

**Why not `io_uring`?** It's the future, but the API is different enough that starting with `epoll` teaches the classical model that most tutorials, textbooks, and codebases use. We can migrate later.

**Connection lifecycle:**

```
listen socket ─── accept ──▶ Connection object (owns the fd)
                                     │
                                     ├─ EPOLLIN  → read into buffer
                                     ├─ EPOLLOUT → drain buffer to socket
                                     └─ EPOLLHUP → destroy Connection
```

Every `Connection` owns its socket fd and its read/write buffers. When the object dies, the fd closes. RAII, no leaks.

### Files (planned)

```
src/
├── main.cpp              ← argv parsing, starts the Reactor
├── reactor.{h,cpp}       ← epoll/kqueue abstraction
├── connection.{h,cpp}    ← per-connection state (fd, buffers, phase)
├── listener.{h,cpp}      ← accept loop
└── log.{h,cpp}           ← tiny structured logger (level, tag, kv pairs)
tests/
├── unit/
│   └── reactor_test.cpp
└── integration/
    └── echo_smoke.py     ← spawns proxy, opens 100 connections, asserts echo
```

### Key decisions to make when starting

- **Buffer strategy:** fixed-size per connection (say 64 KB) or growable? → Start fixed. Simpler. Real proxies use fixed with backpressure.
- **Error handling on the hot path:** exceptions or error codes? → Error codes (`std::expected` in C++23, or a custom `Result<T, Err>` for now). Exceptions have branch-prediction and unwind costs we don't want on the accept loop.
- **Logging:** printf-style or structured? → Structured from day one. Future observability sits on this.

### Done criteria

- [ ] `./switchyard --port 9000` accepts connections.
- [ ] `echo "hi" | nc localhost 9000` returns `hi`.
- [ ] Integration test opens 100 concurrent connections, each echoes 10 messages, all pass.
- [ ] `strace` shows one thread, `epoll_wait` in the main loop.
- [ ] Ctrl-C cleanly closes all connections (no leaked fds — verified with `lsof`).

---

## Rung 2 — HTTP/1.1 Parser + Canned Response

### Goal

On top of Rung 1's reactor: read HTTP/1.1 request bytes, parse into a structured `Request` object (method, path, headers, body). Respond with a hardcoded `200 OK` and a JSON body.

Still not a proxy. This rung makes the HTTP wire format concrete.

### Design

**Parser style:** hand-written state machine, one byte at a time. `llhttp` (nginx/Node.js parser) is the reference to study, but we're writing our own for the learning value.

**States:**
```
REQUEST_LINE  →  HEADER_NAME  →  HEADER_VALUE  →  (more headers?)
                                                      │
                                                      ├─ no  → BODY (if Content-Length > 0) → DONE
                                                      └─ yes → HEADER_NAME
```

**Framing rules we must implement correctly:**
- `\r\n` between lines. Reject bare `\n` and bare `\r` per RFC 9112.
- `Content-Length` (integer, decimal). Reject if it disagrees with what we actually read.
- `Transfer-Encoding: chunked` — deferred to a later rung. For now: reject.
- Header name is case-insensitive. Store lowercased for lookup.
- Request line size cap: 8 KB. Header block size cap: 32 KB. Beyond → `413 Payload Too Large`.

**Response:** hardcoded. `HTTP/1.1 200 OK\r\nContent-Type: application/json\r\nContent-Length: N\r\n\r\n{"proxy":"switchyard","rung":2}`

### Files (added)

```
src/
├── http/
│   ├── parser.{h,cpp}    ← state machine
│   ├── request.{h,cpp}   ← parsed request object
│   └── response.{h,cpp}  ← response builder
tests/
├── unit/
│   ├── http_parser_test.cpp     ← malformed inputs, edge cases
│   └── http_parser_fuzz.cpp     ← libFuzzer harness (nice-to-have)
└── integration/
    └── http_smoke.py            ← curl → assert canned response
```

### Done criteria

- [ ] `curl -v localhost:9000/anything` returns the canned JSON.
- [ ] Parser handles all HTTP methods (GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS).
- [ ] Malformed requests get correct error codes (400, 413, 501).
- [ ] Unit tests cover: split requests across TCP reads (buffer boundary in middle of headers), oversized request line, missing `\r\n`, negative `Content-Length`, duplicate headers.
- [ ] Fuzz harness runs for 60 seconds without crashing.

---

## Rung 3 — Single-Backend Forwarder

### Goal

This is the rung where Switchyard becomes a reverse proxy for the first time. Every request received is forwarded to a hardcoded upstream (e.g. `127.0.0.1:8080`). Whatever the upstream responds, we send back to the client.

### Design

**Config file (YAML or TOML — decision at start of rung):**
```yaml
listen: 0.0.0.0:9000
upstream: 127.0.0.1:8080
```

**Per-request flow:**
```
client → accept → parse Request → open Upstream connection
                                    │
                                    ▼
                              send Request bytes
                                    │
                                    ▼
                              read Response bytes → send to client → close both
```

**Upstream connection lifecycle:** open a fresh TCP connection to the upstream for every request. Slow and wasteful, but simple. Connection pooling lands at Rung 5.

**Both sockets are non-blocking** and registered with the same reactor. A single `Request` object holds references to both connections and progresses through phases:
```
PARSING_REQUEST → CONNECTING_UPSTREAM → SENDING_TO_UPSTREAM
                → READING_FROM_UPSTREAM → SENDING_TO_CLIENT → DONE
```

**Header rewriting:**
- Add `X-Forwarded-For: <client-ip>` (append if header already present).
- Add `X-Forwarded-Proto: http`.
- Remove `Connection: close` from what we forward — decide keep-alive independently.
- Preserve `Host` header (upstream might route on it).

**Error handling:**
| Failure | Response to client |
|---|---|
| Upstream connect fails | 502 Bad Gateway |
| Upstream reset mid-response | Close client connection (nothing else we can do) |
| Upstream times out (not implemented yet — deferred to Rung 7) | — |

### Files (added)

```
src/
├── config/
│   └── config.{h,cpp}          ← parse YAML/TOML, hold the routing table
├── upstream/
│   ├── connection.{h,cpp}      ← outbound TCP connection to backend
│   └── forwarder.{h,cpp}       ← request-lifecycle state machine
tests/
├── unit/
│   ├── config_test.cpp
│   └── forwarder_test.cpp      ← mock upstream, verify header rewriting
└── integration/
    └── proxy_smoke.py          ← spawns upstream + proxy, verifies E2E
```

### Done criteria

- [ ] Config file parses; startup fails loudly if the file is missing or malformed.
- [ ] `curl localhost:9000/foo` reaches the upstream and returns the upstream's response.
- [ ] `X-Forwarded-For` is present in the request the upstream receives.
- [ ] When upstream is down, client gets a proper `502 Bad Gateway` (not a hang, not a crash).
- [ ] Integration test: 100 concurrent requests through the proxy all succeed against a Python `http.server` upstream.
- [ ] p50 latency overhead (proxy vs direct-to-upstream) under 1ms on localhost.

---

## Deferred to Later Rungs

Things we are intentionally *not* solving in Rungs 1-3:

| Concern | Lands at |
|---|---|
| Multiple backends, load balancing | Rung 4 |
| Reusing upstream connections | Rung 5 |
| Detecting dead backends | Rung 6 |
| Timeouts, retries, retry budgets | Rung 7 |
| TLS (HTTPS on client side) | Rung 8 |
| HTTP/2, framed protocol, multiplexing | Rung 9 |
| Hot config swap (the state-first thesis) | Rung 10 |
| Structured request traces + admin API | Rung 11 |
| Shared state across instances (replication) | Rung 12 |
| Chunked transfer encoding | Rung 2.5 (revisit when a real client needs it) |
| WebSocket upgrade | after Rung 9 |

If you find yourself designing for one of the above while building Rungs 1-3, stop. Write down the concern, put it in a `docs/decisions/` note, and come back to it when its rung arrives. Premature design is the biggest tax on systems projects.
