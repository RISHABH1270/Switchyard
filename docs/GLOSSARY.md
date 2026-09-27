# Glossary

Plain-English definitions of every term used in this project. When something is fuzzy, look here first. Add entries as new terms come up — this file grows with the ladder.

---

## Networking basics

**TCP (Transmission Control Protocol)**
Reliable, ordered stream of bytes between two computers. If a packet is lost, TCP retransmits. If packets arrive out of order, TCP reassembles. Every HTTP request rides on top of TCP.

**UDP (User Datagram Protocol)**
Same layer as TCP but unreliable and unordered — fire-and-forget packets. Used by DNS, video streaming, and QUIC (which HTTP/3 rides on).

**Socket**
The programming interface for talking to the network. In POSIX it's just a file descriptor (an integer). `read(fd, buf, len)` reads bytes from the network; `write(fd, buf, len)` sends them.

**Port**
A number (0–65535) that identifies which program on a machine should receive the connection. Web servers usually listen on 80 (HTTP) or 443 (HTTPS).

**File descriptor (fd)**
An integer the kernel hands you when you open something (file, socket, pipe). Every open connection = one fd. Servers can run out of fds (`ulimit -n`), a common production bug.

**Connection**
An established TCP session between two endpoints. Lifecycle: open → send/receive → close. Both sides must close; if one crashes, the other eventually notices.

**NIC (Network Interface Card)**
The hardware that connects a server to the network. Modern NICs do 10/25/100 Gbps and support hardware offloads (TSO, GRO, TLS, checksum).

**DMA (Direct Memory Access)**
Mechanism by which the NIC (or any device) reads/writes RAM directly, without going through the CPU. Modern network I/O relies on DMA for zero-copy paths.

---

## HTTP terms

**HTTP (Hypertext Transfer Protocol)**
The request/response protocol browsers and APIs speak. Versions: HTTP/1.0, 1.1, 2, 3.

**HTTP/1.1**
Text-based. One request per connection at a time. Simple to parse. What we implement in Rung 2.

**HTTP/2**
Binary, framed, multiplexed. Many requests share one TCP connection concurrently via streams. Lands at Rung 9.

**HTTP/3**
HTTP/2 semantics riding on QUIC (over UDP) instead of TCP. Solves head-of-line blocking at the transport layer. Out of scope for v1.

**Request line**
The first line of an HTTP/1.1 request: `GET /login HTTP/1.1`. Method + path + version.

**Headers**
Key-value metadata after the request line. Name is case-insensitive.

**Body**
Optional bytes after the headers. Length given by `Content-Length` or streamed via `Transfer-Encoding: chunked`.

**Status code**
3-digit response code: 200 OK, 404 Not Found, 502 Bad Gateway. 1xx info, 2xx success, 3xx redirect, 4xx client error, 5xx server error.

**Frame (HTTP/2)**
A binary chunk that makes up an HTTP/2 stream. 9-byte header (length, type, flags, stream_id) + payload. Types: HEADERS, DATA, SETTINGS, PING, WINDOW_UPDATE, GOAWAY.

**Stream (HTTP/2)**
A logical bidirectional message flow within a single TCP connection. One connection can carry hundreds of concurrent streams.

**GOAWAY**
An HTTP/2 control frame that tells the peer "no new streams on this connection." Used for graceful shutdown / connection migration.

---

## Proxy concepts

**Proxy**
A server that sits between a client and another server, forwarding requests and responses.

**Forward proxy**
Sits in front of clients (corporate VPN, ad blockers). Server on the other end doesn't know clients are behind a proxy.

**Reverse proxy**
Sits in front of servers. Clients think they're talking directly to a server; really they're talking to the proxy, which routes to backends. What Switchyard is.

**Backend / upstream**
The real server the reverse proxy forwards requests to. Synonyms in this context.

**Route / routing**
The decision about which backend a request goes to, usually based on URL path or `Host` header.

**Load balancing**
The algorithm that picks which backend from a pool. Round-robin, least-connections, adaptive (P2C + PEWMA, latency-aware).

**Health check**
Background probe (`GET /healthz` every 5s) against each backend. Backends failing checks get skipped.

**Connection pooling**
Keep a pool of already-open TCP connections to backends and reuse them. Saves TCP + TLS handshake latency.

**TLS termination**
Handling TLS at the proxy. Browser talks HTTPS to the proxy; the proxy talks plain HTTP to backends inside the private network.

**Timeout**
"If the backend hasn't replied in N seconds, give up." Without them, one slow backend hangs thousands of requests.

**Retry**
"If the backend returned 5xx, try again — maybe a different backend." Paired with a **retry budget** to avoid retry storms.

**Backpressure**
When the proxy realizes it can't push requests to backends fast enough, push back on clients (slow accepts, reject with 503) instead of buffering to OOM.

**Header rewriting**
Adding/removing/changing headers as the proxy forwards. Classic: `X-Forwarded-For`, `X-Forwarded-Proto`.

**Sticky session**
Always send the same client's requests to the same backend (cookie or IP hash). Needed for backends with in-memory session state.

---

## I/O and concurrency

**Blocking I/O**
`read()` sits and waits until data is available. Simple, but ties up a thread per connection. Doesn't scale past ~10k connections.

**Non-blocking I/O**
`read()` returns immediately with `EAGAIN` if no data is ready. You check back later. This is what event loops use.

**Event loop**
A single thread watching many sockets at once. When a socket has data (or capacity), the loop wakes and handles it. Turn-based, single-threaded, extremely efficient.

**epoll**
Linux's classical event-loop primitive. `epoll_ctl` to register sockets, `epoll_wait` to block until something happens. Used by nginx, HAProxy, Envoy, and Cloudflare's Pingora.

**kqueue**
BSD/macOS equivalent of epoll. Same idea, different API. **Not used by Switchyard** — included here for context when discussing other proxies. Switchyard is Linux-only; macOS developers use Docker/Colima.

**`io_uring`**
Newer Linux syscall interface (since 5.1, matured in 5.11+, more so in 6.x). A pair of ring buffers (Submission Queue, Completion Queue) shared between userspace and the kernel. Userspace pushes I/O ops; kernel returns completions. Advantages over epoll:
- **Batching**: many ops submitted with one `io_uring_enter` syscall (or zero, with `IORING_SETUP_SQPOLL`).
- **Multishot ops**: `accept`/`recv` posted once, fires repeatedly.
- **Registered buffers/fds**: `IORING_REGISTER_BUFFERS` pre-pins memory so kernel can DMA directly.
- **True zero-copy send**: `IORING_OP_SEND_ZC` and `SENDMSG_ZC`.
- **File I/O too**: unlike epoll, `io_uring` covers disk ops.

**Submission Queue / Completion Queue (SQ/CQ)**
The two ring buffers at the heart of `io_uring`. SQE = Submission Queue Entry (op you want done). CQE = Completion Queue Entry (result of a completed op).

**`SEND_ZC` (send-zerocopy)**
`io_uring` op that lets the kernel DMA from your userspace buffer directly to the NIC without copying. Completes twice: once when kernel is done reading the buffer (you can now reuse it), once when the send is fully acknowledged.

**Multishot operations**
`io_uring` feature where one submitted op (e.g. `multishot_accept`) keeps producing completions until you explicitly cancel it. Fewer syscalls per event.

**Reactor**
The pattern where an event loop dispatches ready-socket events to handlers. "When socket X is readable, call handler Y." Rung 1's core class.

**C++20 coroutines**
Language-level suspend/resume support. Lets `co_await reactor.recv(fd, buf)` look sequential while actually being asynchronous. Used to write reactor callbacks without callback hell.

**Multithreading**
Multiple OS threads sharing memory. Cheap communication, hard synchronization. Used sparingly in Switchyard (thread-per-core, shared-nothing).

**Thread-per-core**
Pin one thread per CPU core, each running its own reactor. No shared mutable state between them. What HAProxy, Envoy, and Pingora do.

---

## Zero-copy and kernel-assisted I/O

**Memory copy (`memcpy`)**
Moving bytes from one memory location to another. Costs CPU cycles, memory bandwidth, and L1/L2 cache pressure. On a busy proxy, memory copies alone can be 30-50% of total CPU.

**Zero-copy**
Any technique that moves data (from client → proxy → backend) without allocating a userspace buffer and calling `memcpy`. Bytes stay in the kernel or move directly via DMA.

**`writev` / `sendmsg`**
Gather-I/O syscalls. Take a list of `{pointer, length}` pairs (an `iovec`) and write them all in one call. Kernel handles the "merge" without needing you to concatenate in userspace. Used to send `[modified_headers, original_body_view]` in one op.

**`iovec`**
The `struct iovec { void* iov_base; size_t iov_len; }` used by `writev`/`readv`/`sendmsg`.

**`splice()`**
Linux syscall that moves bytes between two file descriptors *without copying to userspace*. Bytes flow kernel-to-kernel via a pipe. Perfect for forwarding request/response bodies.

**`sendfile()`**
Older Linux syscall for moving file → socket without userspace copy. Predecessor of `splice`.

**Zero-copy framing (HTTP/2)**
Parsing and writing HTTP/2 frames without copying payload bytes. Frame headers (9 bytes) built in userspace; payloads represented as `{ptr, len}` views over the original recv buffer.

**`HeaderView`**
Switchyard's zero-copy header representation. `{name_offset, name_length, value_offset, value_length}` — a view into the recv buffer, not a `std::string`. See `src/http/request.h`.

**Registered buffers (`IORING_REGISTER_BUFFERS`)**
`io_uring` feature that pre-pins a set of userspace buffers so the kernel can DMA to/from them without page-table walks. Required for `SEND_ZC`.

---

## TLS and cryptography

**TLS (Transport Layer Security)**
Encryption + authentication layer on top of TCP. Every `https://` URL rides TLS. Current versions: TLS 1.2 (widely deployed), TLS 1.3 (modern default).

**TLS handshake**
The initial exchange (1-2 RTT) where client and server agree on ciphers, exchange keys, and authenticate via certificates. Expensive.

**TLS session resumption**
Skip the full handshake on subsequent connections by presenting a session ticket. Saves 1 RTT and a lot of CPU.

**Session ticket**
An opaque token issued by the server that a client can present to skip a full handshake.

**kTLS (Kernel TLS)**
Linux kernel feature (since 4.13) that moves TLS record framing and encryption/decryption into the kernel. Userspace handles the handshake, then hands the negotiated keys to the kernel via `setsockopt(TCP_ULP, "tls")`. After that, plain `send()` writes get encrypted in-kernel and DMA'd to the NIC. Eliminates the userspace ↔ kernel bounce for every record.

**BoringSSL / OpenSSL**
Cryptographic libraries. BoringSSL is Google's fork of OpenSSL (used by Chrome, Envoy, gRPC). OpenSSL is the mainline. Both support kTLS setup.

**AES-NI**
Intel/AMD CPU instructions that accelerate AES encryption. Any modern proxy relies on them. kTLS uses them from the kernel.

---

## Concurrency and lock-freedom

**Lock / mutex**
A primitive that ensures only one thread accesses shared state at a time. Correct but slow — contention becomes visible above ~50k RPS/core.

**Lock-free**
Data structures that don't use mutexes. Progress is guaranteed for *some* thread at any time. Uses atomic operations (compare-and-swap, atomic increments).

**Wait-free**
Stronger than lock-free. Every thread makes progress in bounded time, regardless of contention. Ideal for hot paths.

**MPMC queue (Multi-Producer Multi-Consumer)**
A queue that multiple threads can push to and pop from concurrently. Lock-free variants (e.g. Vyukov's MPMC) are used for Switchyard's upstream connection pool at Rung 5.

**RCU (Read-Copy-Update)**
Concurrency pattern where readers see a snapshot of state without locking. Writers build a *new* snapshot, then atomically swap a pointer. Readers finish with the old snapshot; a "grace period" ensures nobody is still reading before the old snapshot is freed. Perfect for hot config swap and endpoint tables.

**Per-CPU data**
State stored per-CPU with no cross-CPU synchronization. Reads are cheap (local); writes are cheap (local). Aggregation happens on demand (e.g. summing per-CPU counters to answer a metrics scrape).

**Atomic**
A memory operation guaranteed to be indivisible. `std::atomic<uint64_t>::fetch_add(1, memory_order_relaxed)` is a cheap atomic increment.

**False sharing**
Two variables that logically don't share state end up in the same CPU cache line. Threads updating each variable ping-pong the cache line between cores, killing performance. Avoided by cache-line-aligning per-CPU data.

**Cache line**
The unit of memory transfer between RAM and CPU cache. Usually 64 bytes on x86-64. Struct layouts should be designed with cache lines in mind.

**NUMA (Non-Uniform Memory Access)**
On multi-socket servers, each CPU has "local" RAM (fast) and "remote" RAM (slower). NUMA-aware code pins threads and their data to the same node.

---

## Observability

**Log**
Single line of text saying "this happened." Cheap to write, hard to correlate.

**Metric**
A number that changes over time, aggregated by a metrics system (Prometheus). Example: `switchyard_requests_total`.

**Trace**
Structured record of one specific request through the system, with per-phase timing. Answers "why was *this* request slow." Rung 11.

**Span**
One phase within a trace. A request might span accept, TLS handshake, route decision, upstream connect, upstream response.

**p50, p95, p99, p999**
Percentiles. p99 latency = "99% of requests were faster than this." The tail (p99, p999) is what users notice; averages hide it. **Watch the tail.**

**High-cardinality metric**
A metric with many unique label combinations (e.g. `path` as a label). Explodes storage cost in Prometheus. Switchyard exposes low-cardinality metrics by default.

---

## Systems anti-patterns and failure modes

**Head-of-line blocking (HoL)**
Slow request blocks all requests behind it on the same connection. HTTP/1.1 has request-level HoL; HTTP/2 has TCP-level HoL (one dropped packet stalls all streams); HTTP/3 fixes both.

**Slowloris**
Attack (or slow mobile client) that opens many connections and sends bytes very slowly, tying up server resources. Countered with idle timeouts + per-IP connection caps.

**Thundering herd**
Many clients retry at exactly the same time (e.g. right after an outage recovers), overwhelming the recovered service. Countered with jitter.

**Cold start**
Fresh backend joins the pool but its caches/JIT/connection pools are empty. First N requests are slow. LB should ramp traffic gradually.

**Retry storm**
Partial failure becomes total because every failed request gets retried, multiplying load on the struggling backend. Distributed-systems failure mode.

---

## Reference proxies (for comparison)

**nginx**
The most-deployed reverse proxy. Master + N workers, epoll, C, per-worker state. Config in `nginx.conf`. Reload via SIGHUP forks new workers.

**HAProxy**
L4/L7 load balancer. C, epoll, master-worker mode since 1.8 with seamless reload via `SO_REUSEPORT` + `SCM_RIGHTS` fd passing.

**Envoy**
Modern service-mesh proxy. C++, epoll, thread-per-CPU, xDS dynamic config from a control plane (Istio uses it). Rich filter chain; heavy memory footprint.

**Pingora**
Cloudflare's Rust proxy framework. `tokio` runtime (epoll-based), work-stealing scheduler. Not a drop-in proxy — a library.

**Traefik / Caddy**
Go-based proxies with easy config and auto-TLS. GC-pause tail-latency limits them for extreme-throughput scenarios.

**picohttpparser**
Reference SIMD-accelerated HTTP/1.1 parser used by h2o. Good baseline for what a fast parser looks like.

---

## Kubernetes (for future rungs if Switchyard runs there)

**Pod**
Smallest deployable unit in K8s. One or more containers sharing network namespace + volumes.

**Service**
K8s abstraction that gives a stable virtual IP + DNS name for a set of pods. Routes to pod IPs behind the scenes.

**EndpointSlice**
K8s object listing the actual pod IPs backing a Service, sliced for scalability. What a proxy would watch to know its upstreams.

**Node**
A physical or virtual machine in the K8s cluster. Pods run on nodes.

**Zone / AZ (Availability Zone)**
Cloud-provider concept: an isolated datacenter within a region. Cross-zone traffic costs money (~$0.01/GB per direction on AWS/GCP/Azure).

**Rolling deploy**
K8s update strategy: replace pods one at a time (or in batches) with the new version. Old pods drained via SIGTERM + `terminationGracePeriodSeconds`.

**`terminationGracePeriodSeconds`**
How long K8s waits between SIGTERM and SIGKILL when terminating a pod. Default 30s.

**Sidecar**
A second container in the same pod as the main app. Envoy in Istio is a sidecar per app pod.
