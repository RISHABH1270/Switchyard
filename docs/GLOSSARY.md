# Glossary

Formal definitions for every term used in this project. Where a term has a canonical textbook definition, that is used first; project-specific context (Switchyard's design choices, which rung the term lands on) follows underneath.

---

## Operating system fundamentals

**Kernel**
A kernel is the core part of an operating system. It acts as a bridge between software applications and the hardware of a computer.

The kernel manages system resources — such as the CPU, memory, and devices — ensuring everything works together smoothly and efficiently. All privileged operations (talking to the NIC, allocating memory, scheduling threads) happen inside the kernel.

**User space (Userspace)**
User space is the memory area where all user-mode applications work. It is separated from kernel space to prevent applications from directly accessing critical system resources.

Programs like nginx, your web browser, and Switchyard itself all run in user space with limited privileges. To perform privileged operations they must ask the kernel via system calls.

**System call (syscall)**
A system call is the programmatic way in which a program requests a service from the operating system's kernel. It provides an interface between a running program and the kernel.

Examples include `read()`, `write()`, `open()`, `close()`, `epoll_wait()`, and `io_uring_enter()`. Each system call is a boundary crossing between user space and kernel space, which costs CPU cycles. Reducing syscall count per request is one of Switchyard's central optimisation goals (see Rung 1).

---

## Networking basics

**TCP (Transmission Control Protocol)**
TCP is one of the main protocols of the Internet protocol suite. It provides reliable, ordered, and error-checked delivery of a stream of bytes between two computers communicating over a network.

If a packet is lost in transit, TCP retransmits it. If packets arrive out of order, TCP reassembles them. HTTP, FTP, SMTP, and most application-layer protocols run on top of TCP.

**UDP (User Datagram Protocol)**
UDP is a connectionless communication protocol that provides a simple, unreliable message service. Unlike TCP, UDP does not guarantee delivery, ordering, or duplicate protection.

UDP is faster than TCP because it has no handshake or acknowledgement overhead. Used by DNS, live video streaming, VoIP, and QUIC (which HTTP/3 rides on).

**Socket**
A socket is an endpoint for sending or receiving data across a computer network. It provides the programming interface between an application and the transport-layer protocol (TCP or UDP).

In POSIX systems, a socket is represented as a file descriptor, and standard operations like `read()`, `write()`, and `close()` work on it the same way they work on files.

**Port**
A port is a 16-bit unsigned number (0–65535) that identifies a specific process or service on a networked device. It allows multiple applications on the same host to communicate over the network simultaneously.

Ports combine with an IP address to form a socket address. Well-known ports include 80 (HTTP), 443 (HTTPS), 22 (SSH), and 25 (SMTP).

**File descriptor (fd)**
A file descriptor is a non-negative integer that the kernel uses to identify an open file, socket, pipe, or other I/O resource for a process.

Every open connection consumes one file descriptor. When a process exceeds its file-descriptor limit (`ulimit -n`), new I/O operations fail — a common cause of production outages in networked applications.

**Connection**
A connection is an established communication session between two endpoints over a network, most commonly TCP. It has a lifecycle: open (handshake) → send / receive data → close (teardown).

Both endpoints must close their side. If one side crashes without closing, the other eventually detects it via a keep-alive probe or timeout.

**NIC (Network Interface Card)**
A NIC is a hardware component that connects a computer to a network. It handles the physical and data-link layers of network communication.

Modern server NICs run at 10, 25, 100, or 200 Gbps and include hardware offloads for tasks such as TCP segmentation (TSO), receive coalescing (GRO), TLS record encryption, and checksum computation.

**DMA (Direct Memory Access)**
DMA is a hardware feature that allows I/O devices such as a NIC or disk controller to read from or write to system memory (RAM) directly, without involving the CPU.

DMA is what makes zero-copy I/O possible: bytes can move from disk → memory → NIC without the CPU touching them.

---

## HTTP terms

**HTTP (Hypertext Transfer Protocol)**
HTTP is an application-layer protocol used to transmit hypermedia documents, such as HTML, between a client and a server. It follows a request/response model over a reliable transport (TCP for HTTP/1.x and HTTP/2, QUIC for HTTP/3).

Versions: HTTP/1.0, HTTP/1.1 (most widely deployed), HTTP/2 (binary, multiplexed), HTTP/3 (runs on QUIC over UDP).

**HTTPS (HTTP Secure)**
HTTPS is HTTP transported inside a TLS-encrypted tunnel. It uses the same HTTP semantics as plain HTTP but adds confidentiality (encryption), integrity, and server authentication via TLS.

**HTTP/1.1**
Text-based version of HTTP. One request per connection at a time (pipelining is broken in practice). Simple to parse. What we implement in Rung 2.

**HTTP/2**
Binary, framed, and multiplexed version of HTTP. Many requests share one TCP connection concurrently via streams. Lands at Rung 9.

**HTTP/3**
HTTP/2 semantics running on QUIC (over UDP) instead of TCP. Solves head-of-line blocking at the transport layer. Out of scope for v1.

**Request line**
The first line of an HTTP/1.1 request: `GET /login HTTP/1.1`. Method + path + protocol version.

**Header**
Case-insensitive key-value metadata that appears between the request/response line and the body. Examples: `Host`, `Content-Length`, `User-Agent`.

**Body**
Optional payload bytes that follow the headers. Length is signalled by `Content-Length` or streamed via `Transfer-Encoding: chunked`.

**Status code**
A 3-digit integer returned in an HTTP response indicating the result of the request. Categories: 1xx (informational), 2xx (success), 3xx (redirection), 4xx (client error), 5xx (server error).

**Frame (HTTP/2)**
The unit of communication in HTTP/2. Each frame has a 9-byte header (length, type, flags, stream identifier) followed by a payload. Types include `HEADERS`, `DATA`, `SETTINGS`, `PING`, `WINDOW_UPDATE`, and `GOAWAY`.

**Stream (HTTP/2)**
A logical bidirectional message flow within a single TCP connection. One HTTP/2 connection can carry hundreds of concurrent streams.

**GOAWAY**
An HTTP/2 control frame sent by a peer to indicate it will not accept new streams on the connection. Used for graceful shutdown and connection migration.

---

## Proxy concepts

**Proxy**
A server that sits between a client and another server, forwarding requests in one direction and responses in the other.

**Forward proxy**
A proxy that sits in front of clients — for example, a corporate VPN or content filter. The remote server is unaware that clients are behind a proxy.

**Reverse proxy**
A proxy that sits in front of one or more origin servers. Clients believe they are talking directly to the origin; in reality they are talking to the proxy, which routes each request to an appropriate backend. Switchyard is a reverse proxy.

**Backend / upstream**
The origin server to which a reverse proxy forwards requests. The two terms are used interchangeably in this project.

**Route / routing**
The decision, made by the proxy, of which backend a given request should be sent to. Typically based on URL path, `Host` header, or method.

**Load balancing**
The algorithm the proxy uses to select a backend from a pool of equivalent ones. Common strategies: round-robin, least-connections, latency-aware (e.g. Power-of-Two-Choices with PEWMA).

**Health check**
A periodic probe (for example, `GET /healthz` every 5 seconds) sent by the proxy to each backend. Backends failing checks are removed from the load-balancing pool until they recover.

**Connection pooling**
Reusing already-established TCP (and optionally TLS) connections to backends instead of opening a new one per request. Saves the handshake round-trips on every subsequent request.

**TLS termination**
Handling TLS at the reverse proxy. The client speaks HTTPS to the proxy; the proxy speaks plain HTTP to backends over the private network.

**Timeout**
A rule that aborts a request if a response has not been received within N seconds. Without timeouts, a single slow backend can hang thousands of concurrent client requests.

**Retry**
If a backend returns a server error (5xx) or fails to respond, the proxy may resend the request — potentially to a different backend. Must be paired with a **retry budget** to prevent retry storms.

**Backpressure**
The situation in which the proxy cannot push requests to backends as fast as clients are sending them. A well-behaved proxy responds by slowing new accepts or returning 503, rather than buffering unbounded amounts of memory until it OOMs.

**Header rewriting**
Adding, removing, or altering HTTP headers as the proxy forwards a request or response. Classic examples: `X-Forwarded-For` (records the original client IP), `X-Forwarded-Proto` (records the original scheme).

**Sticky session**
Consistently routing a given client's requests to the same backend (via a cookie or IP hash). Required for backends that hold in-memory session state.

---

## I/O and concurrency

**Blocking I/O**
A model in which an I/O operation (e.g. `read()`) suspends the calling thread until the operation completes. Simple to program, but each concurrent connection requires its own thread — which does not scale past ~10 000 connections.

**Non-blocking I/O**
A model in which an I/O operation returns immediately with `EAGAIN`/`EWOULDBLOCK` if it cannot make progress. The application must poll or use an event-notification mechanism to find out when the operation is ready. All modern high-performance servers use non-blocking I/O.

**Event loop**
A programming construct that waits for and dispatches events. A single thread watches many sockets simultaneously; when a socket becomes readable or writable, the loop invokes the associated handler. Turn-based, single-threaded, extremely efficient.

**epoll**
epoll is a Linux kernel system call for scalable I/O event notification. `epoll_ctl` registers file descriptors of interest, and `epoll_wait` blocks until one or more of them is ready. Used by nginx, HAProxy, Envoy, and Cloudflare's Pingora.

**kqueue**
kqueue is BSD and macOS's equivalent of epoll — a scalable event-notification mechanism. Same idea, different API. **Not used by Switchyard**; included for context. Switchyard is Linux-only and macOS developers use Docker/Colima.

**`io_uring`**
`io_uring` is a Linux kernel interface for asynchronous I/O introduced in kernel 5.1 (2019). It uses a pair of ring buffers — a Submission Queue and a Completion Queue — shared between user space and the kernel. Applications place I/O requests on the Submission Queue and read results from the Completion Queue, with dramatically fewer system calls than epoll.

Advantages over epoll:
- **Batching**: many operations submitted with one `io_uring_enter` (or zero, with `IORING_SETUP_SQPOLL`).
- **Multishot ops**: `accept` / `recv` posted once, fire repeatedly.
- **Registered buffers/fds**: `IORING_REGISTER_BUFFERS` pre-pins memory so the kernel can DMA directly.
- **True zero-copy send**: `IORING_OP_SEND_ZC` and `SENDMSG_ZC`.
- **File I/O too**: unlike epoll, `io_uring` covers disk operations.

**Submission Queue / Completion Queue (SQ / CQ)**
The two ring buffers at the heart of `io_uring`. An SQE (Submission Queue Entry) describes an operation to perform; a CQE (Completion Queue Entry) describes the result.

**`SEND_ZC` (send-zerocopy)**
An `io_uring` operation that lets the kernel DMA bytes from a user-space buffer directly to the NIC without copying. It completes twice: once when the kernel has finished reading the buffer (so the caller may reuse it), and again when the send is fully acknowledged.

**Multishot operations**
An `io_uring` feature where a single submitted operation (e.g. `multishot_accept`) keeps producing completions until the caller explicitly cancels it. Reduces the number of system calls per event.

**Reactor**
The reactor design pattern is a concurrency model in which an event demultiplexer waits for events on multiple resources and dispatches them to handlers. "When socket X is readable, call handler Y." Rung 1's core class implements this pattern on top of `io_uring`.

**C++20 coroutines**
Language-level support in C++20 for functions that can suspend and resume execution. Coroutines let `co_await reactor.recv(fd, buf)` read as sequential code while actually being asynchronous under the hood — used in Switchyard to avoid callback-style code around `io_uring`.

**Multithreading**
Multithreading is the ability of a CPU (or a single core) to provide multiple threads of execution concurrently, supported by the operating system. Threads within a process share memory but each has its own stack and register state.

**Thread-per-core**
An architectural pattern in which one thread is pinned to each CPU core, each running its own event loop with no shared mutable state between threads. Used by HAProxy, Envoy, and Pingora. Switchyard adopts it starting at Rung 5.

---

## Zero-copy and kernel-assisted I/O

**Memory copy (`memcpy`)**
`memcpy` is a standard C library function that copies a block of bytes from one memory location to another. Each byte crosses the memory bus, consuming CPU cycles, memory bandwidth, and L1/L2 cache. On a busy reverse proxy, `memcpy` alone can account for 30–50% of total CPU time.

**Zero-copy**
Zero-copy refers to a class of techniques in which data is transferred between the source and destination without being copied through a user-space buffer. Bytes either stay inside the kernel or move directly via DMA between hardware and memory.

**`writev` / `sendmsg`**
POSIX gather-I/O system calls that accept a list of `{pointer, length}` pairs (an `iovec`) and write them all in a single call. The kernel handles the "merge," so user space does not need to concatenate the pieces itself. Switchyard uses `writev` to send `[modified_headers, original_body_view]` in one operation.

**`iovec`**
The POSIX structure `struct iovec { void *iov_base; size_t iov_len; }`, used with `writev`, `readv`, and `sendmsg` to describe a scatter/gather list of memory regions.

**`splice()`**
A Linux system call that moves bytes between two file descriptors without copying them to user space. Data flows kernel-to-kernel via an intermediate pipe. Ideal for forwarding request/response bodies through a proxy.

**`sendfile()`**
A Linux system call that copies data from a file descriptor (typically a file) to another (typically a socket) inside the kernel. The predecessor of `splice`, and still used for serving static files.

**Zero-copy framing (HTTP/2)**
Parsing and constructing HTTP/2 frames without ever copying the payload bytes. Frame headers (9 bytes) are built in user space; payloads are represented as `{pointer, length}` views into the original receive buffer.

**`HeaderView`**
Switchyard's zero-copy representation of an HTTP header: `{name_offset, name_length, value_offset, value_length}` — a view into the receive buffer, never a `std::string`. See `src/http/request.h`.

**Registered buffers (`IORING_REGISTER_BUFFERS`)**
An `io_uring` feature that pre-pins a set of user-space buffers with the kernel, so subsequent operations can DMA into or out of them without page-table walks. Required for `SEND_ZC`.

---

## TLS and cryptography

**TLS (Transport Layer Security)**
TLS is a cryptographic protocol that provides secure communication over a computer network. It provides three guarantees: confidentiality (encryption), integrity (tamper detection), and authentication (identity verification via X.509 certificates).

TLS 1.2 is the most widely deployed version; TLS 1.3 (RFC 8446, 2018) is the modern default. Every `https://` URL rides TLS.

**TLS handshake**
The initial protocol exchange (one or two round-trips) in which the client and server negotiate a cipher suite, exchange key material, and authenticate via certificates. Handshakes are computationally expensive relative to the steady-state data transfer that follows.

**TLS session resumption**
An optimisation that skips the full handshake on subsequent connections between the same client and server, by presenting a previously issued session ticket or session ID. Saves at least one round-trip and a significant amount of CPU.

**Session ticket**
An opaque, server-issued token that a client can present on a later connection to resume a previous TLS session without a full handshake.

**kTLS (Kernel TLS)**
kTLS is a Linux kernel feature (introduced in kernel 4.13, 2017) that moves TLS record framing and symmetric encryption/decryption from user space into the kernel. User space still performs the handshake; the negotiated keys are then handed to the kernel via `setsockopt(SOL_TCP, TCP_ULP, "tls")`. After this, plain `send()` writes are encrypted in-kernel and DMA'd directly to the NIC, eliminating the user-space ↔ kernel record bounce.

**BoringSSL / OpenSSL**
BoringSSL and OpenSSL are cryptographic libraries providing TLS implementations. OpenSSL is the mainline project used by most of the software ecosystem; BoringSSL is Google's fork used by Chrome, Envoy, and gRPC. Both support kTLS setup.

**AES-NI (Advanced Encryption Standard – New Instructions)**
AES-NI is a set of x86 CPU instructions introduced by Intel and AMD that accelerate AES encryption and decryption in hardware. Any modern TLS-terminating proxy relies on them; kTLS invokes them from inside the kernel.

---

## Concurrency and lock-freedom

**Mutex (mutual exclusion)**
A mutex is a synchronisation primitive used to protect a shared resource from concurrent access by multiple threads. Only one thread can hold a mutex at a time; others wait until it is released.

Correct but potentially slow: contention becomes visible in profilers once request rates exceed ~50 000 per core.

**Lock-free**
Lock-free is a property of a concurrent algorithm that guarantees at least one thread will make progress at any given moment, without using traditional locks. Achieved via atomic operations such as compare-and-swap.

**Wait-free**
Wait-free is a stronger progress guarantee than lock-free: every thread completes any given operation in a bounded number of steps, regardless of contention. Ideal for hot paths where tail latency matters.

**MPMC queue (Multi-Producer Multi-Consumer)**
A concurrent queue that multiple threads may push to and multiple threads may pop from simultaneously. Lock-free implementations (for example Vyukov's MPMC queue) are used in Switchyard's global upstream connection pool at Rung 5.

**RCU (Read-Copy-Update)**
RCU is a synchronisation mechanism used extensively in the Linux kernel. Readers access shared data without any locking; writers create a new copy of the data structure and atomically swap a pointer to it. Old readers finish with the previous copy, and a grace period ensures no reader is still accessing the old copy before it is freed.

RCU is a good fit for hot-swappable configuration, endpoint tables, and any data structure that is read very frequently and written rarely.

**Per-CPU data**
State that is stored per-CPU with no cross-CPU synchronisation. Both reads and writes are local (and therefore cheap). Aggregation (for example, summing per-CPU counters for a metrics scrape) happens on demand.

**Atomic**
An operation is atomic if it appears to occur instantaneously from the perspective of other threads — it cannot be interleaved with another operation. Modern hardware provides atomic instructions such as compare-and-swap and atomic add. Example: `std::atomic<uint64_t>::fetch_add(1, std::memory_order_relaxed)`.

**False sharing**
False sharing occurs when two variables that are logically independent are placed in the same CPU cache line. Threads updating each variable cause the cache line to bounce between cores, degrading performance. Avoided by cache-line-aligning per-CPU data.

**Cache line**
A cache line is the unit of memory transfer between main memory (RAM) and CPU cache. On x86-64, a cache line is 64 bytes. Data-structure layout should take cache-line boundaries into account.

**NUMA (Non-Uniform Memory Access)**
NUMA is a memory-architecture design used in multi-socket servers, where each CPU has its own local memory that it can access faster than memory attached to another CPU. NUMA-aware software pins threads and their working sets to the same NUMA node.

---

## Observability

**Log**
A log is a timestamped record of a discrete event ("this happened"). Logs are cheap to write but expensive to correlate across services.

**Metric**
A metric is a numeric measurement that changes over time, typically aggregated by a monitoring system such as Prometheus. Example: `switchyard_requests_total`.

**Trace**
A trace is a structured record of a single request as it moves through a system, capturing per-phase timing. Traces answer the question "why was *this specific* request slow?" Rung 11.

**Span**
A span represents a single operation or phase within a trace. A request through Switchyard may include spans for accept, TLS handshake, route decision, upstream connect, and upstream response.

**p50, p95, p99, p999**
Percentile latency values. For example, p99 latency = "99% of requests were faster than this value." The tail (p99, p999) is what users actually perceive; simple averages hide it. **Watch the tail.**

**High-cardinality metric**
A metric label whose value space is very large (for example, `path` as a label). High-cardinality labels explode storage cost in Prometheus. Switchyard exposes low-cardinality metrics by default.

---

## Systems anti-patterns and failure modes

**Head-of-line blocking (HoL)**
Head-of-line blocking occurs when the first in-flight request delays every request queued behind it on the same connection. HTTP/1.1 has request-level HoL; HTTP/2 has TCP-level HoL (one dropped packet stalls all streams); HTTP/3 fixes both by moving to QUIC.

**Slowloris**
Slowloris is a denial-of-service attack (or, unintentionally, a slow mobile client) in which many connections are opened and data is sent at an extremely low rate, tying up server resources. Mitigated with idle timeouts and per-IP connection caps.

**Thundering herd**
Thundering herd is a failure mode in which many clients retry at exactly the same moment — for example immediately after an outage recovers — overwhelming the recovered service. Mitigated with jittered retry timing.

**Cold start**
Cold start occurs when a new backend instance joins the pool with empty caches, empty JIT code caches, and empty connection pools. Its first N requests are slow. A load balancer should ramp traffic to a cold instance gradually.

**Retry storm**
A retry storm is a distributed-systems failure mode in which a partial backend failure becomes total because every failed request is retried, multiplying the load on the already-struggling backend. Prevented with retry budgets and circuit breakers.

---

## Reference proxies (for comparison)

**nginx**
The most-deployed reverse proxy. Master + N workers, epoll, C, per-worker state. Config in `nginx.conf`. Reload via `SIGHUP` forks new workers and drains old ones.

**HAProxy**
Widely deployed L4/L7 load balancer. C, epoll, master-worker mode since 1.8 with seamless reload via `SO_REUSEPORT` and `SCM_RIGHTS` file-descriptor passing.

**Envoy**
Modern service-mesh proxy. C++, epoll, thread-per-CPU, dynamic configuration via xDS from an external control plane (Istio, Consul, etc.). Rich HTTP filter chain; comparatively heavy memory footprint.

**Pingora**
Cloudflare's Rust proxy framework. Runs on the `tokio` async runtime (epoll-based), work-stealing scheduler. Distributed as a library rather than a drop-in proxy binary.

**Traefik / Caddy**
Go-based reverse proxies with easy configuration and automatic TLS certificate management. Garbage-collection pauses limit tail-latency stability at extreme throughput.

**h2o**
C-based HTTP server and proxy known for very low overhead and the reference `picohttpparser` implementation.

**picohttpparser**
A SIMD-accelerated HTTP/1.1 parser used by h2o. Serves as a baseline for what a fast, zero-copy HTTP parser looks like.

---

## Kubernetes (for future rungs if Switchyard runs there)

**Pod**
A pod is the smallest deployable unit in Kubernetes. It consists of one or more containers that share a network namespace and storage volumes.

**Service**
A Kubernetes Service is an abstraction that defines a stable virtual IP and DNS name for a dynamic set of pods, routing traffic to the currently healthy pod IPs behind the scenes.

**EndpointSlice**
An EndpointSlice is a Kubernetes API object that lists the actual pod IPs backing a Service, sliced into groups for scalability. A proxy watches EndpointSlices to know its current set of upstreams.

**Node**
A Node in Kubernetes is a physical or virtual machine that runs pods. Nodes are managed by the control plane.

**Zone / Availability Zone (AZ)**
A zone is an isolated failure domain within a cloud-provider region — typically a separate data-centre with its own power and cooling. Cross-zone network traffic is billed by the major cloud providers at roughly $0.01 per GB per direction.

**Rolling deploy**
A Kubernetes update strategy in which pods are replaced with the new version one at a time (or in small batches). Old pods are drained via `SIGTERM` and `terminationGracePeriodSeconds` before the new pods come online.

**`terminationGracePeriodSeconds`**
A field on a Kubernetes pod spec specifying how long the kubelet waits between sending `SIGTERM` and `SIGKILL` when terminating a pod. Default is 30 seconds.

**Sidecar**
A sidecar is a supporting container running alongside the main application container within the same pod. In Istio, an Envoy sidecar runs beside every application pod to handle service-mesh traffic.
