# Glossary

Plain-English definitions of every term used in this project. When something is fuzzy, look here first. Add entries as new terms come up — this file grows with the ladder.

---

## Networking basics

**TCP (Transmission Control Protocol)**
A way for two computers to send a *reliable, ordered* stream of bytes to each other over the internet. If a packet is lost, TCP retransmits it. If packets arrive out of order, TCP reassembles them. Every HTTP request rides on top of TCP.

**UDP (User Datagram Protocol)**
Same layer as TCP, but *unreliable and unordered* — fire-and-forget packets. Used by DNS, video streaming, and QUIC (which HTTP/3 rides on). Not used in Switchyard until much later, if ever.

**Socket**
The programming interface your code uses to talk to the network. In C++/POSIX it's just a file descriptor (an integer). `read(fd, buf, len)` on a socket fd reads bytes from the network; `write(fd, buf, len)` sends them.

**Port**
A number (0–65535) that identifies *which program* on a machine should get the connection. Machines have one IP address but 65,535 ports. Web servers usually listen on 80 (HTTP) or 443 (HTTPS).

**File descriptor (fd)**
An integer the kernel gives you when you open something (file, socket, pipe). Every open connection = one fd. Servers can run out of fds (`ulimit -n`) which is a common production bug.

**Connection**
An established TCP session between two endpoints. Has a lifecycle: open → send/receive bytes → close. Both sides must close; if one side crashes, the other eventually notices.

---

## HTTP terms

**HTTP (Hypertext Transfer Protocol)**
The request/response protocol browsers and APIs speak. Comes in versions: HTTP/1.0, 1.1, 2, 3. Rides on top of TCP (1.x, 2) or QUIC (3).

**HTTP/1.1**
Text-based, one request per connection at a time. Simple to parse. What we implement in Rung 2.

**HTTP/2**
Binary, framed, multiplexed. Many requests share one TCP connection concurrently (streams). Faster in most cases but complex to parse. Lands at Rung 9.

**HTTP/3**
HTTP/2 semantics, but rides on QUIC (over UDP) instead of TCP. Solves head-of-line blocking at the transport layer. Out of scope for Switchyard v1.

**Request line**
The first line of an HTTP/1.1 request: `GET /login HTTP/1.1`. Method + path + version.

**Headers**
Key-value metadata after the request line: `Host:`, `User-Agent:`, `Content-Type:`, etc. Name is case-insensitive.

**Body**
Optional bytes after the headers. Present on POST/PUT typically. Length is either given by `Content-Length` or streamed via `Transfer-Encoding: chunked`.

**Status code**
The 3-digit number in an HTTP response: 200 OK, 404 Not Found, 502 Bad Gateway, etc. Categories: 1xx info, 2xx success, 3xx redirect, 4xx client error, 5xx server error.

---

## Proxy concepts

**Proxy**
A server that sits between a client and another server. Forwards requests one direction and responses the other.

**Forward proxy**
Sits in front of *clients*. Used to control/filter outbound traffic (e.g. corporate VPN, ad blockers). The server on the other end doesn't know clients are behind a proxy.

**Reverse proxy**
Sits in front of *servers*. Clients think they're talking directly to a server; really they're talking to the proxy, which routes to one of many backends. What Switchyard is.

**Backend / upstream**
The real server the reverse proxy forwards requests to. "Backend" and "upstream" are synonyms in this context.

**Route / routing**
The decision the proxy makes about which backend a request goes to, usually based on URL path or `Host` header.

**Load balancing**
The algorithm that picks *which* backend from a pool of equivalent ones. Simple: round-robin. Better: least-connections. Best: adaptive (P2C + PEWMA, latency-aware).

**Health check**
A background probe (`GET /healthz` every 5 seconds, say) the proxy runs against each backend. Backends failing checks are skipped by the load balancer.

**Connection pooling**
Instead of opening a fresh TCP connection to the backend for every request, keep a pool of already-open connections and reuse them. Saves TCP handshake + TLS handshake latency.

**TLS termination**
Handling TLS (the S in HTTPS) at the proxy. The browser talks encrypted HTTPS to the proxy; the proxy talks plain HTTP to backends on the private network. Backends don't need certificates.

**Timeout**
A rule like "if the backend hasn't replied in 5 seconds, give up." Without timeouts, one slow backend hangs thousands of client requests.

**Retry**
"If the backend returned a 5xx, try again — maybe against a different backend." Must be paired with a **retry budget** (max % of requests that can be retries) to avoid retry storms.

**Retry storm**
A partial outage that becomes total because every failed request gets retried, multiplying load on the already-struggling backend, which makes it fail more, which triggers more retries. A well-known distributed systems failure mode.

**Backpressure**
When the proxy realizes it can't push requests to backends fast enough, it *pushes back* on clients (slows accepting new requests, or rejects with 503) instead of buffering forever in memory until it crashes. Backpressure is what turns a chaotic overload into a graceful degradation.

**Header rewriting**
Adding/removing/changing headers as the proxy forwards a request. Classic examples: `X-Forwarded-For` (records the original client IP), `X-Forwarded-Proto` (records the original scheme).

**Sticky session**
When a proxy always sends the same client's requests to the same backend (based on a cookie or IP hash). Needed for backends that hold in-memory session state. Fragile — one backend dying strands its sessions.

---

## State-first concepts (the Switchyard thesis)

**State**
Anything the proxy remembers between requests: which backends are healthy, which users have hit their rate limit, which TLS sessions can be resumed, what the current config says. The core Switchyard thesis is that this state is what causes production pain today.

**Hot config reload / hot config swap**
Updating the proxy's configuration (routes, backends, timeouts) without restarting the process and without dropping in-flight requests or long-lived connections. nginx's "graceful reload" isn't fully hot — it spawns new workers and drains old ones. Switchyard aims for truly atomic swaps.

**RCU (Read-Copy-Update)**
A concurrency pattern where readers see a snapshot of state without locking. Writers build a *new* snapshot, then atomically swap a pointer. Perfect for hot config: request handlers read the config lock-free; a config-update thread swaps in the new one atomically. Widely used in the Linux kernel.

**Sharded state**
Splitting state across processes or nodes so no single instance holds all of it. Instead of one giant rate-limit table, each key hashes to a shard.

**Replicated state**
State that lives on multiple nodes so any of them can serve any request consistently. Different from sharding. Both patterns show up in Rung 12.

**Introspectable state**
State that can be *read at runtime* via an admin API — e.g. `GET /admin/state/upstreams` returns the current list of backends and their health. Necessary for debugging production without SSHing in.

---

## I/O and concurrency

**Blocking I/O**
When a call like `read()` sits and waits until data is available. Simple, but ties up a thread per connection. Doesn't scale past ~10k connections.

**Non-blocking I/O**
`read()` returns immediately with `EAGAIN` if no data is ready. You have to check back later. This is what event loops use.

**Event loop**
A single thread that watches many sockets at once. When a socket has data to read (or capacity to write), the loop wakes up and handles it. Turn-based, single-threaded, extremely efficient.

**epoll**
Linux's event-loop primitive. `epoll_ctl` to add sockets, `epoll_wait` to block until something happens. Used by nginx, HAProxy, Envoy.

**kqueue**
The BSD/macOS equivalent of epoll. Same idea, different API.

**io_uring**
A newer Linux syscall interface (since 5.1). More performant than epoll but different mental model — submission queue + completion queue. Might replace epoll in Switchyard down the line.

**Reactor**
The pattern where an event loop dispatches ready-socket events to handlers. "When socket X is readable, call handler Y." Rung 1's core class.

---

## Observability

**Log**
A single line of text saying "this happened." Cheap to write, hard to correlate across services.

**Metric**
A number that changes over time, aggregated by a metrics system (Prometheus, etc.). Examples: `switchyard_requests_total`, `switchyard_active_connections`.

**Trace**
A structured record of *one specific request* moving through the system, with timing for each phase. Answers "why was *this* request slow." The core of Switchyard's Rung 11 observability.

**Span**
One phase within a trace. A single request might have spans for accept, TLS handshake, route decision, upstream connect, upstream response.

**p50, p95, p99, p999**
Percentiles. p99 latency = "99% of requests were faster than this." The tail (p99, p999) is what users actually notice — averages hide it. Watching the tail is a habit of good backend engineers.

---

## Anti-patterns and failure modes

**Head-of-line blocking (HoL)**
When a slow request blocks all the ones behind it on the same connection. HTTP/1.1 has request-level HoL (pipelining broken); HTTP/2 has TCP-level HoL (one dropped packet stalls all streams); HTTP/3 fixes both.

**Slowloris**
An attack (or accidental behavior from slow mobile clients) where a client opens many connections and sends bytes very slowly, tying up server resources. Countered with idle timeouts and per-IP connection caps.

**Thundering herd**
When many clients simultaneously retry at the exact same time (say, right after an outage recovers), overwhelming the recovered service. Countered with jitter — spreading retries over a random interval.

**Cold start**
When a fresh backend instance joins the pool but its caches/JIT/connection pools are all empty. First N requests to it are slow. LB algorithms should ramp traffic gradually.
