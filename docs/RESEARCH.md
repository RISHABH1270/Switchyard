# Research: The State of Reverse-Proxy Performance in 2026

This document is Switchyard's honest survey of the L7 reverse-proxy landscape. It exists so we don't ship claims we can't defend, and so contributors can see exactly where each incumbent stops improving and why Switchyard's design choices matter.

Every performance claim in this document is either (a) cited from public benchmarks / vendor documentation, or (b) explicitly flagged as "to be measured by Switchyard's own benchmark harness in `bench/`."

---

## 1. Why revisit reverse proxies now?

Three hardware and kernel shifts have quietly outrun the design assumptions of nginx (2004), HAProxy (2000), and Envoy (2016):

1. **NIC line rates** went from 1 Gbps → 10 → 25 → 100 → 200 Gbps in twenty years. Every generation exposes new bottlenecks in the userspace/kernel boundary that were invisible when the network was the bottleneck.
2. **CPU core counts** went from 2-4 → 64-128 per socket. Shared-nothing thread-per-core is now table stakes; anything relying on a single accept mutex or a global connection pool is CPU-starved.
3. **The kernel added new I/O primitives** — `io_uring` (5.1, 2019), kTLS (4.13, 2017), `SEND_ZC` (6.0, 2022), multishot ops (5.19+) — that nginx, HAProxy, and Envoy have not fundamentally re-architected around.

The result is a growing gap between what modern hardware and kernels can deliver and what mainline proxies actually achieve. That gap is measured in wasted CPU, wasted memory bandwidth, higher tail latency, and larger fleets.

---

## 2. Landscape at a glance

| Proxy | First release | Language | I/O model | TLS strategy | Config | Notable strength | Notable weakness |
|---|---|---|---|---|---|---|---|
| **nginx** | 2004 | C | epoll, master + N workers | Userspace (OpenSSL); optional kTLS partial | Static + reload | Ubiquity, docs, ecosystem | Config reload spawns new workers; no `io_uring`; body-buffering defaults |
| **HAProxy** | 2000 | C | epoll, master-worker (2.0+) | Userspace | Static + seamless reload | Seamless reload via SCM_RIGHTS; excellent L4 | No `io_uring`; limited scripting (Lua only) |
| **Envoy** | 2016 | C++11 | epoll, thread-per-CPU | Userspace (BoringSSL) | Dynamic (xDS) | Rich filter chain; service mesh integration | 50-150 MB/instance baseline; higher CPU per request; no `io_uring` |
| **Traefik** | 2015 | Go | epoll via Go netpoller | Userspace | Dynamic, K8s-native | Easy config, auto-TLS | GC pauses hurt tail latency at scale |
| **Caddy** | 2015 | Go | epoll via Go netpoller | Userspace | Dynamic, easy | Auto-HTTPS, extensible | GC pauses; not built for extreme throughput |
| **Pingora** | 2022 (OSS 2024) | Rust | `tokio` (epoll) | Userspace (BoringSSL) | Programmatic (library) | Memory safety; work-stealing | Not a drop-in proxy; still epoll-based |
| **h2o** | 2014 | C | epoll | Userspace | Static | SIMD parsing (`picohttpparser`); low overhead | Smaller ecosystem; less production hardening |
| **HAProxy Fusion / Enterprise** | commercial | C | epoll | Userspace + hardware offload | Dynamic + REST | Best-in-class L4 perf | Not OSS |

**Nothing in this table is `io_uring`-native. Nothing in this table is kTLS-first. That's the gap.**

---

## 3. Deep dive: where each incumbent leaves CPU on the table

### 3.1 nginx

**Architecture:**
```
┌──────────────┐    fork/exec
│   master     │──────────────▶ worker (epoll, single-thread)
│  (nginx.conf)│──────────────▶ worker
│              │──────────────▶ worker  ← N workers, one per CPU
└──────────────┘                worker
```
- Master reads config; workers own sockets. Per-worker state (no cross-worker sharing).
- Reload = master forks new workers with new config; old workers drain.

**Low-level gaps:**

1. **Epoll-based, no `io_uring`.** As of nginx 1.27 (2024) there is no mainline `io_uring` backend. Third-party patches exist but are not upstream.
2. **Body buffering default.** `proxy_request_buffering on` (default) writes the request body to disk before forwarding upstream. Kills streaming; adds copies; delays upstream response.
3. **Per-worker connection pools.** Upstream keepalive pools are per-worker. On a 32-core box you have 32 pools; connections are not shared. More TCP + TLS handshakes than necessary.
4. **DNS resolution per worker.** No shared cache; TTL races. Requires `resolver` directive and even then is imperfect.
5. **Written in pre-C11 C.** Memory safety is a manual discipline. Annual CVE cadence.
6. **Header parsing copies into buffer pools.** Not zero-copy in the strict sense; header names/values live in pool-allocated buffers, not views over the recv buffer.

**Public perf datapoint:** nginx reference benchmarks in 2019 showed ~110k RPS/core on a small keep-alive workload; that number is essentially unchanged in 2024. Cloudflare's Pingora blog stated ~50% CPU reduction over their previous nginx-based stack for the same workload — a hint at how much slack exists.

---

### 3.2 HAProxy

**Architecture:**
```
┌──────────────┐    seamless reload (SCM_RIGHTS + SO_REUSEPORT)
│   master     │──────────────▶ worker (epoll, single-thread)
│ (haproxy.cfg)│──────────────▶ worker  ← workers share the listening
└──────────────┘                worker    socket via kernel
```
- Since 1.8: passes listening socket fd from old worker to new via `SCM_RIGHTS` and `SO_REUSEPORT`. **Zero dropped connections on config reload within the same pod.**
- Multi-threaded worker mode (2.0+): one worker with N threads, each thread pinned to a core.

**Low-level gaps:**

1. **Epoll-based.** Same story as nginx.
2. **Config reload doubles memory transiently** during reload (old and new worker both alive).
3. **Lua-only scripting.** Extending L7 logic without recompiling means embedded Lua, which has runtime cost.
4. **Sticky tables (in-memory session tables) are per-instance.** Behind a L4 LB, users see inconsistent state.
5. **No first-class dynamic service discovery.** Data Plane API is bolt-on.
6. **Written in pre-C11 C.** Same safety concerns as nginx.

**Public perf datapoint:** HAProxy 2.4 benchmarks show ~200k RPS/core on small HTTP requests, ahead of nginx but still on epoll. HAProxy's own benchmarks (2023) note the seamless reload feature does not reduce steady-state performance; it just avoids the reload cliff.

---

### 3.3 Envoy

**Architecture:**
```
┌────────────────────┐
│  Envoy Process     │
│  ┌──────────────┐  │
│  │ main thread  │  │   xDS config from control plane
│  ├──────────────┤  │
│  │ worker 0     │  │   epoll loop, thread-local pool, thread-local stats
│  ├──────────────┤  │
│  │ worker N     │  │
│  └──────────────┘  │
└────────────────────┘
```
- Multi-threaded within one process (thread-per-CPU).
- Dynamic config via xDS gRPC push from an external control plane (Istio, Consul, custom).
- Hot restart: old Envoy passes sockets + shared-memory stats to new Envoy via `SCM_RIGHTS`.

**Low-level gaps:**

1. **Epoll-based.** As of Envoy 1.30 (2024), no `io_uring` backend upstream.
2. **50-150 MB baseline memory per instance.** Rich feature set costs memory even at idle. Compare to nginx (~5-15 MB) and HAProxy (~5-30 MB).
3. **Filter chain is a virtual-call chain.** Each HTTP filter is invoked via virtual dispatch per request. CPU overhead measurable at high QPS.
4. **20-40% higher CPU per request than nginx** for equivalent workloads (widely cited in benchmark comparisons; verify in `bench/`).
5. **Buffer management copies.** Envoy's `Buffer::Instance` abstraction is not strictly zero-copy; copies happen at filter boundaries.
6. **xDS complexity.** Powerful but the config surface is enormous; misconfigurations are a common outage cause.
7. **Sidecar tax.** In Istio, every app pod runs an Envoy sidecar; per-hop latency 1-3ms, per-pod memory 50-150 MB.

**Public perf datapoint:** Envoy vs nginx head-to-head benchmarks (e.g. by Netflix, LinkedIn) consistently show Envoy at ~1.3-1.6× CPU for the same workload, in exchange for richer features and dynamic config. Envoy's own docs acknowledge this tradeoff.

---

### 3.4 Pingora (Cloudflare)

**Architecture:**
- Rust, `tokio` runtime, work-stealing scheduler.
- Library, not a daemon: you write your proxy as a Rust program using Pingora's building blocks.

**Low-level gaps:**

1. **Still epoll-based.** `tokio` uses `mio`, which uses epoll on Linux. Pingora inherits this.
2. **Not a drop-in.** You must write Rust to use it. Ops teams can't just deploy it.
3. **Newer, smaller ecosystem** than nginx/HAProxy/Envoy.

**What Pingora gets right:**
- Memory safety (Rust).
- Work-stealing scheduler outperforms nginx's static worker model on skewed workloads.
- Cloudflare's own perf claim: **~50% CPU reduction** vs their previous nginx-based stack for the same production traffic (public blog, 2022).

**Why this matters for Switchyard:** Pingora proves the market rewards a from-scratch modern proxy. It also proves there's still room to go further — because Pingora didn't touch `io_uring` or kTLS.

---

### 3.5 h2o

**Architecture:**
- C, epoll.
- Ships `picohttpparser` — a SIMD-accelerated HTTP/1.1 parser using SSE4 or SSE4.2 to scan header bytes in 16-byte chunks.
- `picohttpparser` is the reference to study for fast parsing.

**Public perf datapoint:** h2o has historically outperformed nginx on synthetic benchmarks by 20-40% on parse-heavy workloads, largely due to `picohttpparser`. In production it hasn't displaced nginx due to smaller ecosystem.

---

## 4. The three levers Switchyard pulls

Switchyard's thesis is not novel research — every technique below is already in the Linux kernel and used by parts of the stack. What's new is combining all three in one general-purpose L7 proxy.

### 4.1 `io_uring`-native reactor

**What:** submit every socket op through the `io_uring` submission queue. No epoll fallback on Linux. Use multishot `accept`/`recv` to reduce syscall count. Use registered buffers so the kernel can DMA directly.

**Expected win:**
- **Syscalls per request:** epoll baseline ~4-6 syscalls (`epoll_wait`, `read`, `write`, `close`). `io_uring` amortizes to ~1 (`io_uring_enter`, often skipped entirely via `SQPOLL` kernel thread).
- **Throughput:** independent benchmarks (Josh Triplett, Axboe et al.) show `io_uring` beats epoll by 2-3× on small-message workloads.
- **Tail latency:** less syscall overhead = less scheduling jitter = tighter p99.

**Precedent:** ScyllaDB (database), Glommio (Rust runtime), Netflix's video-CDN experiments. No mainline L7 proxy has adopted it yet.

**Risks:**
- Requires Linux 6.1+ for stable feature set.
- API surface still expanding — will need to track kernel changes.
- Linux-only by design. macOS developers work through Docker/Colima or a Linux VM. No cross-platform reactor abstraction — the thesis primitives are Linux-only, so paying for portability buys nothing.

### 4.2 Zero-copy request/response path

**What:** HTTP parser produces `HeaderView { offset, length }` records over the original recv buffer, never `std::string`. Header rewriting uses a small side-buffer (~200 bytes); body forwarding uses `writev` or `splice`. Body bytes never enter userspace when we can help it.

**Expected win:**
- **Memory bandwidth:** eliminates 2 of the 4 copies per request. On 10 Gbps traffic, saves ~20 GB/s of memory bandwidth.
- **L1/L2 cache pressure:** parsed views stay in the recv buffer's cache line; no extra allocation walks.
- **CPU:** 15-25% of total CPU in copy-heavy proxies goes to `memcpy`. Zero-copy targets 2-5%.

**Precedent:** `picohttpparser` demonstrates zero-copy parsing at the HTTP/1.1 layer. `splice()` has been standard in nginx for `sendfile` scenarios but not for body forwarding in the general case.

**Risks:**
- Zero-copy parsers are harder to write and read. Cognitive tax.
- `splice()` has edge cases (limited to socket-to-pipe-to-socket).
- Registered buffers add complexity to buffer lifetime management.

### 4.3 kTLS-first

**What:** TLS handshakes run in userspace (BoringSSL/OpenSSL). Immediately after the handshake, negotiated keys are handed to the kernel via `setsockopt(TCP_ULP, "tls")`. From that point, plain `send()` writes are encrypted in-kernel using AES-NI and DMA'd to the NIC.

**Expected win:**
- **CPU on TLS-heavy traffic:** 20-40% reduction (per Netflix's kTLS deployment, 2016).
- **Copies eliminated:** removes the userspace-crypto-buffer bounce for every TLS record.
- **Composes with `SEND_ZC`:** encrypted output can DMA directly from the plaintext buffer.

**Precedent:** Netflix has deployed kTLS at massive scale for video since 2016; documented ~2× throughput on TLS-heavy workloads. nginx has partial kTLS support (enable-tls-support flag); HAProxy has partial support. Nobody is kTLS-first from day one.

**Risks:**
- kTLS support in OpenSSL/BoringSSL is uneven (fewer ciphers supported).
- Debugging TLS issues is harder when records are in kernel space.
- Some middleboxes (that terminate TLS) don't play well with kTLS.

---

## 5. Enhancement matrix — gap → technique → expected win

| Incumbent's gap | Switchyard's technique | Expected win | Verification |
|---|---|---|---|
| Epoll's 4-6 syscalls per request | `io_uring` with multishot + SQPOLL | 2-3× throughput; -30% p99 | `bench/rung1_echo_qps.sh` |
| 3-4 copies of every payload byte | `HeaderView` + `writev` + `splice` | -20% CPU; -50% memory bandwidth | `bench/rung3_forward_qps.sh` |
| Userspace TLS record framing | kTLS from day one | -20-40% CPU on TLS | `bench/rung8_tls_qps.sh` |
| Per-worker connection pools | Global lockless MPMC queue | Fewer upstream conns; less handshake overhead | `bench/rung5_pool.sh` |
| Locks / global stats mutexes | Per-CPU counters, RCU config, wait-free hot path | Linear scaling to 64+ cores | `bench/rung11_scaling.sh` |
| Nginx-style header pool copies | Zero-copy `HeaderView` | Parse <200 ns; no heap alloc per request | `bench/rung2_parse_ns.sh` |
| Envoy filter-chain virtual calls | Compile-time filter chain (C++20 templates or CRTP) | -10-15% CPU vs Envoy | `bench/rung9_h2.sh` |
| Config reload trauma (nginx) | RCU-based atomic swap | Zero dropped requests on reload | `bench/rung10_reload.sh` |

Every row of this table is a design commitment. Every row must be defended by a benchmark in `bench/` before Rung 12 is called "done."

---

## 6. Benchmark methodology

The value of this project is not just the code — it is a **reproducible benchmark suite** that anyone can run against a canned upstream and compare Switchyard vs nginx vs HAProxy vs Envoy on identical hardware.

**Setup rules (documented in `bench/README.md` when we get there):**

1. **Isolated cores.** `isolcpus` + `nohz_full` + `rcu_nocbs` on the benchmark cores.
2. **Same kernel, same NIC, same NUMA node** for proxy and load generator (or documented split).
3. **Same upstream.** A minimal Rust or C canned-response server that never becomes the bottleneck.
4. **Warm-up + long runs.** 30 seconds warm-up, 5 minutes measured, three repetitions.
5. **Full percentile output.** Not just mean — `p50, p95, p99, p999, max`.
6. **CPU accounting.** `perf stat` for cycles, cache-misses, syscalls per request.
7. **All configs published.** nginx.conf, haproxy.cfg, envoy.yaml, switchyard.toml — all in the repo, no cherry-picking.

**Workloads to run:**
- **Small keep-alive requests** — the classic "N k RPS/core" number.
- **Large body pass-through** — 1 MB body, measures copy overhead.
- **TLS handshake storm** — new connections/sec, measures TLS setup cost.
- **HTTP/2 streaming** — long-lived streams with intermittent DATA frames.
- **Mixed workload** — 80% keep-alive, 10% cold, 10% large body (closer to real traffic).

**Presentation:** each rung's benchmark script prints a Markdown table:

```
| Proxy       | RPS/core | p50    | p99    | p999   | CPU%  |
|-------------|---------:|-------:|-------:|-------:|------:|
| nginx       |  110000  | 0.4ms  | 1.8ms  | 4.2ms  |  100% |
| HAProxy     |  200000  | 0.3ms  | 1.4ms  | 3.6ms  |   95% |
| Envoy       |   75000  | 0.6ms  | 2.5ms  | 6.0ms  |  135% |
| Switchyard  |  <TBD>   | <TBD>  | <TBD>  | <TBD>  |  <TBD>|
```

If Switchyard doesn't beat the target row, we do not ship the claim.

---

## 7. Prior art and required reading

**Papers:**
- Axboe, J. "Efficient IO with io_uring" (2019). https://kernel.dk/io_uring.pdf
- Belay et al. "IX: A Protected Dataplane Operating System for High Throughput and Low Latency" (OSDI 2014).
- Marinos et al. "Network Stack Specialization for Performance" (SIGCOMM 2014).

**Kernel docs:**
- `Documentation/networking/tls.rst` — kTLS setup and cipher support.
- `io_uring(7)` manpage — core interface.
- `IORING_SETUP_SQPOLL` — kernel-thread submission for zero-syscall submission.

**Existing implementations to read:**
- `picohttpparser` (h2o) — SIMD HTTP parsing.
- `liburing` examples — reactor patterns.
- Netflix's kTLS Freenas patches — production kTLS deployment lessons.
- Pingora's public architecture posts — modern proxy design in a memory-safe language.

**Benchmarks to reproduce:**
- Cloudflare Pingora vs nginx blog post (2022).
- Envoy vs nginx benchmarks (multiple, various companies).
- io_uring vs epoll microbenchmarks (Axboe, various).

---

## 8. What Switchyard is not

To keep the pitch honest, we should state clearly what Switchyard is *not* trying to be:

- **Not a service mesh.** No control plane, no xDS, no sidecar model. That's Envoy's job.
- **Not a WAF.** No web application firewall features in v1.
- **Not a CDN.** No caching, no edge features. That's Varnish / Cloudflare / Fastly.
- **Not a Kubernetes ingress in v1.** May become one later; v1 is a general-purpose L7 proxy.
- **Not cross-platform.** Linux 6.1+ only. macOS developers use Docker/Colima; there is no `kqueue` fallback.
- **Not a drop-in replacement for nginx configs.** Config format is our own; migration path is a nice-to-have, not a v1 feature.

Every "not" is a decision to spend effort on the thesis instead of feature-checkbox parity.

---

## 9. Open questions (to be answered as the ladder progresses)

- Does `io_uring` `SEND_ZC` interact cleanly with kTLS? Nobody has publicly documented this combination at scale.
- Can we get useful HTTP/2 flow-control accounting without breaking zero-copy discipline?
- What's the right story for graceful drain during a K8s rolling deploy, without falling into the "connection continuity across nodes" trap?
- Should the config format be TOML, YAML, or custom? What's the RCU story for reload atomicity?
- Do C++20 coroutines generate acceptable code with GCC 13 / Clang 16, or do we hit compiler bugs?

These become entries in `docs/decisions/` as we answer them.

---

## 10. Why this document exists

Every ambitious systems project starts with a slide claiming to be faster than the incumbents. Most of them are wrong, because the author didn't understand the incumbents well enough to know where they actually lose. This document is Switchyard's insurance policy against that failure mode. If a design choice in `docs/ARCHITECTURE.md` doesn't map to a gap identified here, either the gap is missing or the design is speculative.

**Contributions to this document are especially welcome.** Corrections, missing citations, and refined benchmark data will make the whole project more defensible.
