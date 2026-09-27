# Architecture

> **Scope of this document:** Rungs 1-3 only.
>
> Rungs 4-12 are intentionally *not* designed here. Systems-level design decisions made months in advance almost always turn out wrong once real code hits the kernel. We'll write a fresh design section for each rung as we approach it, informed by what we've learned building the previous rung.

---

## Guiding Principles

These shape every design decision below. They come directly from the low-level performance thesis: `io_uring`-native, zero-copy, kTLS-first.

1. **Zero-copy discipline from Rung 1.** Every design choice must ask "does this force a memory copy on the hot path?" If yes, redesign until no. Copies are the single largest CPU sink in a modern proxy.
2. **`io_uring`-native, Linux-only.** No epoll fallback, no `kqueue` fallback, no portable-reactor abstraction. Switchyard runs on Linux. macOS developers use Docker or Colima to get a Linux dev environment. An abstraction layer for cross-platform I/O pays for nothing when the entire thesis (`io_uring`, kTLS, `SEND_ZC`, `splice`) is Linux-specific.
3. **State lives per-CPU or in RCU snapshots.** No mutexes on the hot path, ever. Every shared read is either a per-CPU load or a lock-free pointer load. Writes take slower paths (RCU grace period, per-CPU aggregation).
4. **The hot path must be observable.** Every request eventually emits a structured trace with phase boundaries (accept → parse → route → forward → response). We start noting these boundaries from Rung 1, even if we only `printf` today.
5. **Fail loudly in development, gracefully in production.** Assertion-heavy debug builds. Explicit error propagation on the hot path via `std::expected<T, ProxyErr>`; no exceptions.
6. **Testable from Rung 1.** Every rung ships with unit + integration tests and a micro-benchmark. No rung is "done" until its `make bench` output beats a documented baseline.

---

## Target Environment

- **OS:** Linux 6.1+ only (`io_uring` maturity, kTLS reliability, `SEND_ZC` support). No cross-platform build. macOS developers work through Docker/Colima or a Linux VM — dev-loop is `docker run --rm -it -v $PWD:/work -w /work switchyard-dev bash`.
- **Compiler:** GCC 13+ or Clang 16+ with C++20 (coroutines required).
- **Build:** CMake 3.20+.
- **Test framework:** GoogleTest (unit) + Python integration harness (spawn proxy, curl, assert).
- **Bench framework:** `wrk2`, `h2load`, plus a custom harness that pins cores and controls the kernel scheduler.
- **Dependencies for Rungs 1-3:** `liburing` (only on Linux). No Boost/ASIO/libevent — the point is to touch the syscalls directly.

---

## Rung 1 — TCP Echo Server (`io_uring`-native)

### Goal

Accept TCP connections on a configurable port. Read bytes from the client. Write the same bytes back. Close cleanly. Handle 10k+ concurrent clients from a single thread using `io_uring`.

**Not a proxy yet.** This rung exists to make `io_uring`, connection lifecycle, and zero-copy discipline concrete before HTTP is on top.

### Design

**I/O model:** single-threaded event loop using `io_uring`. Wrapped in a small `Reactor` class so unit tests can substitute a `MockReactor` — no other production implementation.

**Why `io_uring` (not epoll)?** The thesis rests on it. Every rung after this depends on submission-queue-batching, multishot ops, and `SEND_ZC`. Starting with epoll would mean rewriting the reactor at Rung 5 or 8. Doing it right now costs one extra week and saves months later.

**Why not threads-per-connection?** Falls over at 10k connections. Not the goal.

**Why not one big thread pool?** Rung 1 stays single-threaded so we can *see* where blocking hurts. Multi-threading arrives at Rung 5 when connection pools go global.

**Connection lifecycle:**

```
listen socket ─── multishot_accept ──▶ Connection { fd, recv_buf, send_buf }
                                              │
                                              ├─ multishot_recv → echo path
                                              ├─ SEND_ZC → completion → free buf
                                              └─ close on EOF/error
```

Every `Connection` owns its socket fd and its recv/send buffers via RAII. Buffers come from a per-CPU pool, not `malloc` per request.

### `Reactor` interface (both backends must satisfy)

```cpp
class Reactor {
public:
  // Submit an accept op that yields when a connection is ready.
  awaitable<AcceptResult> accept(int listen_fd);

  // Submit a recv op that yields with either bytes-received or EOF/error.
  awaitable<RecvResult> recv(int fd, std::span<std::byte> buf);

  // Submit a send op. Returns bytes-written; retries on EAGAIN internally.
  awaitable<SendResult> send(int fd, std::span<const std::byte> buf);

  // Run the reactor loop; returns when told to stop.
  void run();
  void stop();
};
```

`IoUringReactor` is the only production implementation. `MockReactor` (in `tests/`) satisfies the same interface for unit tests.

**Coroutines** are how we get sequential-looking code without callback hell:

```cpp
task<void> echo_loop(Reactor& r, int fd) {
  std::array<std::byte, 65536> buf;
  while (true) {
    auto rcv = co_await r.recv(fd, buf);
    if (rcv.eof) break;
    co_await r.send(fd, {buf.data(), rcv.bytes});
  }
  close(fd);
}
```

### Files (planned)

```
src/
├── main.cpp              ← argv parsing, starts the Reactor
├── reactor/
│   ├── reactor.h         ← interface + awaitables
│   └── io_uring.cpp      ← only production implementation
├── net/
│   ├── connection.h/cpp  ← per-connection state (fd, buffers, phase)
│   └── listener.h/cpp    ← accept loop wrapper
├── mem/
│   └── buffer_pool.h/cpp ← per-CPU buffer arena (fixed 64 KB slabs)
└── log/
    └── log.h/cpp         ← tiny structured logger
tests/
├── unit/
│   └── reactor_test.cpp
└── integration/
    └── echo_smoke.py     ← spawns proxy, opens 100 conns, asserts echo
bench/
└── rung1_echo_qps.sh     ← measures accept+echo QPS single-core
```

### Key decisions to make when starting

- **Buffer strategy:** fixed 64 KB per connection from a per-CPU arena. No growable buffers on the hot path.
- **Error handling:** `std::expected<T, ProxyErr>` return type. No exceptions.
- **Coroutine allocator:** custom promise allocator using per-CPU arena. Avoid default `operator new`.
- **Registered buffers:** use `io_uring`'s `IORING_REGISTER_BUFFERS` on Linux to pin buffers for zero-copy DMA.
- **Backpressure:** when send queue depth exceeds N, stop submitting recvs on that fd.
- **Logging:** structured (level, tag, kv pairs). All observability sits on this.

### Done criteria

- [ ] `./switchyard --port 9000` accepts connections via `io_uring`.
- [ ] `echo "hi" | nc localhost 9000` returns `hi`.
- [ ] Integration test: 10,000 concurrent connections each echoing 10 messages, all pass.
- [ ] `strace -c` shows `io_uring_enter` dominating; almost no per-op syscalls.
- [ ] `perf stat` shows fewer syscalls per request than an epoll baseline (target: 3-5× fewer).
- [ ] Ctrl-C cleanly closes all connections (no leaked fds — verified with `lsof`).
- [ ] `make bench` reports single-core QPS ≥ 2× a naive epoll echo server.

---

## Rung 2 — HTTP/1.1 Parser + Canned Response (Zero-Copy)

### Goal

On top of Rung 1's reactor: read HTTP/1.1 request bytes, parse into structured views (method, path, headers, body) *without copying header bytes into new strings*. Respond with a hardcoded `200 OK` + JSON body.

Still not a proxy. This rung makes the HTTP wire format concrete and locks in zero-copy discipline before we add forwarding.

### Design

**Parser style:** hand-written state machine, one byte at a time. `picohttpparser` (used by h2o) is the reference to study; we're writing our own for the learning value and to control exactly how views are constructed.

**Zero-copy header representation:**

```cpp
struct HeaderView {
  uint16_t name_offset;   // offset into recv buffer
  uint16_t name_length;
  uint16_t value_offset;
  uint16_t value_length;
};

struct RequestView {
  Method method;                    // enum, parsed
  uint16_t path_offset;
  uint16_t path_length;
  Version version;                  // enum
  std::array<HeaderView, 64> headers;
  uint8_t header_count;
  const std::byte* buffer_base;     // points into the Connection's recv buffer
  size_t body_offset;               // where body starts in buffer
  size_t body_length;               // Content-Length or 0 for chunked-deferred
};
```

**Nothing is copied.** All header names/values are `{buffer_base + offset, length}` views. Case-insensitive lookup is done via `strncasecmp` on the view, no allocation.

**States:**
```
REQUEST_LINE  →  HEADER_NAME  →  HEADER_VALUE  →  (more headers?)
                                                      │
                                                      ├─ no  → BODY (if Content-Length > 0) → DONE
                                                      └─ yes → HEADER_NAME
```

**Framing rules we must implement correctly:**
- `\r\n` between lines. Reject bare `\n` and bare `\r` per RFC 9112.
- `Content-Length` decimal integer. Reject if it disagrees with what we read.
- `Transfer-Encoding: chunked` — deferred. For now: reject with 501.
- Header name is case-insensitive. We do NOT normalize; we store the view and compare with `strncasecmp` on lookup.
- Request line size cap: 8 KB. Header block size cap: 32 KB. Beyond → 413.

**Response:** canned. Built once at startup into a `constexpr` byte array. `send()` returns a view into it. No allocation per response.

### Files (added)

```
src/
├── http/
│   ├── parser.h/cpp    ← state machine, produces RequestView
│   ├── request.h       ← RequestView + HeaderView definitions
│   └── response.h/cpp  ← canned response builder
tests/
├── unit/
│   ├── parser_test.cpp        ← malformed inputs, buffer-boundary splits
│   └── parser_fuzz.cpp        ← libFuzzer harness
└── integration/
    └── http_smoke.py
bench/
└── rung2_parse_ns.sh          ← nanoseconds per parse; target: <200 ns for typical request
```

### Done criteria

- [ ] `curl -v localhost:9000/anything` returns the canned JSON.
- [ ] Parser handles GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS.
- [ ] Malformed requests get 400, 413, 501.
- [ ] Unit tests cover: split requests across TCP reads (buffer boundary mid-header), oversized request line, missing `\r\n`, negative `Content-Length`, duplicate headers.
- [ ] Zero heap allocations per request on the hot path (verified with `perf record` — no `malloc` in flame graph).
- [ ] `make bench` reports parse time <200 ns for a typical 5-header request.
- [ ] Fuzz harness runs 60 seconds without crashing.

---

## Rung 3 — Single-Backend Forwarder (writev / splice)

### Goal

Switchyard becomes a reverse proxy for the first time. Every request received is forwarded to a hardcoded upstream (e.g. `127.0.0.1:8080`). Whatever the upstream responds, we send back to the client — **without copying the body bytes through userspace**.

### Design

**Config file (TOML):**
```toml
listen = "0.0.0.0:9000"
upstream = "127.0.0.1:8080"
```

**Per-request flow:**
```
client → accept → parse RequestView → open upstream connection
                                       │
                                       ▼
                                 build modified header block
                                 (small alloc, ~200 bytes)
                                       │
                                       ▼
                                 writev(upstream_fd, [headers, body_view])
                                       │
                                       ▼
                                 recv upstream response header
                                       │
                                       ▼
                                 splice(upstream_fd → pipe → client_fd)
                                       │      (body never enters userspace)
                                       ▼
                                 close both, or hold for keep-alive
```

**Header rewriting** happens in a small userspace buffer (~200 bytes) because we do modify a few headers:
- Add `X-Forwarded-For: <client-ip>` (append if header already present)
- Add `X-Forwarded-Proto: http`
- Strip `Connection: close`
- Preserve `Host`

The body — potentially megabytes — is forwarded via `writev` (for immediate small bodies already in recv buffer) or `splice` (for streaming large bodies). It never lives in a userspace buffer we own.

**Upstream connection:** open fresh TCP connection per request. Slow and wasteful. Pooling lands at Rung 5.

**Both sockets** are registered with the same reactor. A `Forwarder` coroutine owns the state machine:

```cpp
task<void> forward(Reactor& r, ClientConn client) {
  auto req = co_await parse(client);
  auto upstream = co_await connect_upstream(r, cfg.upstream);
  auto modified_headers = build_forward_headers(req);
  co_await writev(upstream, {modified_headers, req.body_view()});
  auto resp_headers = co_await read_response_headers(upstream);
  co_await client.send(resp_headers);
  co_await splice_body(upstream, client, resp_headers.content_length);
}
```

**Error handling:**

| Failure | Response to client |
|---|---|
| Upstream connect fails | 502 Bad Gateway |
| Upstream reset mid-response | Close client connection |
| Upstream timeout | Deferred to Rung 7 |

### Files (added)

```
src/
├── config/
│   └── config.h/cpp        ← TOML parse, routing table
├── upstream/
│   ├── connection.h/cpp    ← outbound TCP connection
│   └── forwarder.h/cpp     ← per-request state machine coroutine
tests/
├── unit/
│   ├── config_test.cpp
│   └── forwarder_test.cpp  ← mock upstream, verify header rewriting
└── integration/
    └── proxy_smoke.py
bench/
└── rung3_forward_qps.sh    ← QPS + p99 vs direct-to-upstream; target: overhead < 5%
```

### Done criteria

- [ ] Config file parses; startup fails loudly if missing/malformed.
- [ ] `curl localhost:9000/foo` reaches upstream and returns upstream's response.
- [ ] `X-Forwarded-For` present in request the upstream receives.
- [ ] When upstream is down, client gets `502 Bad Gateway` (no hang, no crash).
- [ ] Integration test: 1000 concurrent requests through the proxy all succeed against a Python `http.server` upstream.
- [ ] p50 latency overhead (proxy vs direct-to-upstream) under 200 µs on localhost.
- [ ] p99 latency overhead under 1 ms on localhost.
- [ ] `perf record` shows no body bytes touched by userspace (splice used correctly).

---

## Deferred to Later Rungs

Things intentionally *not* solved in Rungs 1-3:

| Concern | Lands at |
|---|---|
| Multiple backends, LB algorithms | Rung 4 |
| Reusing upstream connections | Rung 5 |
| Detecting dead backends | Rung 6 |
| Timeouts, retries, retry budgets | Rung 7 |
| TLS + kTLS | Rung 8 |
| HTTP/2 framing (zero-copy) | Rung 9 |
| RCU-based hot config swap | Rung 10 |
| Wait-free per-CPU observability | Rung 11 |
| Reproducible benchmarks vs nginx/HAProxy/Envoy | Rung 12 |
| Chunked transfer encoding | Rung 2.5 (revisit when a real client needs it) |
| WebSocket upgrade | after Rung 9 |

If you find yourself designing for one of the above while building Rungs 1-3, stop. Write it down in `docs/decisions/`, come back to it when its rung arrives. Premature design is the biggest tax on systems projects.
