<div align="center">

<img src="./assets/banner.svg" alt="Switchyard" width="100%"/>

<br/>
<br/>

[![License: MIT](https://img.shields.io/badge/License-MIT-34D399?style=for-the-badge&labelColor=10B981)](LICENSE)
[![C++20](https://img.shields.io/badge/C%2B%2B-20-60A5FA?style=for-the-badge&logo=cplusplus&logoColor=white&labelColor=3B82F6)](https://en.cppreference.com/w/cpp/20)
[![Status](https://img.shields.io/badge/Status-Rung%200%20Planning-FB923C?style=for-the-badge&labelColor=F97316)](docs/ARCHITECTURE.md)
[![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20macOS-A78BFA?style=for-the-badge&logo=linux&logoColor=white&labelColor=8B5CF6)](#)
[![io_uring](https://img.shields.io/badge/io__uring-native-22D3EE?style=for-the-badge&labelColor=06B6D4)](docs/RESEARCH.md)
[![kTLS](https://img.shields.io/badge/kTLS-first-F472B6?style=for-the-badge&labelColor=EC4899)](docs/RESEARCH.md)

</div>

---

## The Problem

**Reverse proxies have hit a CPU ceiling.**

nginx, HAProxy, and Envoy all share a lineage of design decisions made when 1 Gbps was fast and TLS was expensive to offload. On modern hardware — 100 Gbps NICs, servers with 64+ cores, HTTP/2 and mTLS everywhere — that lineage now bleeds CPU in three specific places:

| Where CPU leaks | Why |
|---|---|
| **Syscall overhead** | All three still use `epoll`. Every request costs 4-6 syscalls. `io_uring` (Linux 5.1+) can amortize this to nearly one. |
| **Userspace memory copies** | Each request byte gets memcpy'd 3-4 times: kernel → recv buffer → parsed struct → send buffer → kernel. On a busy proxy that's 30-50% of total CPU. |
| **Userspace TLS record framing** | Encryption bytes bounce between userspace crypto libraries and kernel sockets. Kernel TLS (kTLS) can eliminate that bounce and go direct to the NIC. |

The result: a modern box that could push 100 Gbps of proxied traffic on paper delivers 30-50 Gbps in practice. The CPU is busy moving bytes, not serving requests. At scale this becomes fleet size × 2, latency × 2, cost × 2.

---

## The Thesis

Switchyard is a reverse proxy designed to eliminate all three of those CPU leaks from the ground up:

1. **`io_uring`-native reactor** — no epoll fallback on Linux. Every socket op flows through a shared submission queue. Multishot accept/recv where supported.
2. **Zero-copy request/response path** — HTTP parser produces `HeaderView { offset, length }` records over the original recv buffer, never `std::string`. Forwarding uses `writev`, `splice`, and `io_uring SEND_ZC`. Body bytes never enter userspace when we can help it.
3. **kTLS-first** — TLS record framing runs in the kernel from Rung 8 onward, not as a bolt-on. Handshakes happen in userspace (BoringSSL/OpenSSL), but everything after the handshake goes direct: userspace → kernel → NIC.

Nothing here is speculative — every technique exists in the Linux kernel today and is used by parts of the stack (Netflix's kTLS for video, Cloudflare's io_uring experiments, various zero-copy patents). What's missing is a general-purpose L7 reverse proxy that combines all three from day one.

**The measurable claim Switchyard is committing to:**

> Same job as nginx (HTTP/1.1 + HTTP/2 + TLS + LB + health + retries + observability). **30-50% less CPU** on identical hardware and workload. **Half the p99 latency**. Head-to-head benchmarks published in `bench/` and reproducible by anyone.

---

## Mental Model

A reverse proxy is a mailroom clerk — receives requests, picks the right backend, forwards them, brings replies back. Switchyard's difference is how few times it *touches* each byte:

```
   Client                          Backend
     │                                ▲
     │  ─ TCP + TLS ─▶                │  ─ TCP + TLS ─
     │                                │
     ▼                                │
   ┌───────────────────────────────────────────────┐
   │                Switchyard                     │
   │                                               │
   │   io_uring reactor  ─── one shared SQ/CQ      │
   │   Zero-copy parser  ─── views, not strings    │
   │   writev / splice   ─── kernel does the merge │
   │   kTLS              ─── crypto in the kernel  │
   └───────────────────────────────────────────────┘
```

Same features as any reverse proxy. Different implementation discipline underneath.

---

## The Build Ladder

Each rung is understandable before starting the next. The **rungs stay the same** as any competent proxy would need — but the **standard for each rung** is sharpened by the thesis:

| Rung | Milestone | Sharpened standard |
|-----:|-----------|--------------------|
| 1  | TCP echo server | `io_uring`-native reactor (`kqueue` fallback for macOS dev only) |
| 2  | HTTP/1.1 parser + canned response | Zero-copy `HeaderView` — no `std::string` for headers |
| 3  | Single-backend forwarder | `writev` for header + body forward; `splice` where legal |
| 4  | Multi-backend + round-robin LB | Wait-free (RCU) endpoint table |
| 5  | Upstream connection pool | Global lockless MPMC queue, not per-worker |
| 6  | Health checks | Same as anyone else |
| 7  | Timeouts + retries with budget | Same as anyone else |
| 8  | TLS termination | **kTLS from day one** (handshake in userspace, records in kernel) |
| 9  | HTTP/2 | Zero-copy frame handling; `HEADERS`/`DATA` payloads as views |
| 10 | Hot config reload | RCU-based atomic config swap |
| 11 | Observability | Wait-free per-CPU counters, aggregated on read |
| 12 | Benchmarks + comparisons | Reproducible harness in `bench/`; head-to-head vs nginx/HAProxy/Envoy |

**Rungs 1-3** = a terrible-but-working reverse proxy. ~3 weeks.
**Rungs 1-8** = a proxy you could put in front of real traffic. ~4-6 months.
**Rungs 1-12** = the full thesis with defensible benchmarks. 8-12 months.

---

## Why C++20

- Direct access to Linux primitives (`io_uring`, `splice`, `kTLS` setsockopt) without a runtime in the way.
- Zero-cost abstractions and manual memory control matter on a zero-copy hot path.
- C++20 coroutines make the `io_uring` submission/completion loop readable without callback hell.
- Mature crypto ecosystem (BoringSSL, OpenSSL) with native bindings.

**Why not Rust?** Rust would be my second choice. But `io_uring` bindings in Rust (`tokio-uring`, `glommio`) are still in flux, and Cloudflare's Pingora (Rust) chose `tokio` which is `epoll`-based. C++ lets us go straight to `liburing` and skip abstraction layers.

**Why not Go?** GC pauses hurt tail latency. Fine for many things, not for a proxy claiming half the p99 of the incumbents.

---

## Repo Layout (planned)

```
switchyard/
├── README.md                    ← you are here
├── LICENSE
├── docs/
│   ├── ARCHITECTURE.md          ← deep plan for current rung
│   ├── GLOSSARY.md              ← plain-English definitions
│   ├── RESEARCH.md              ← comparison vs nginx/HAProxy/Envoy/Pingora
│   └── decisions/               ← ADRs for non-obvious choices
├── src/                         ← C++20 source (from Rung 1)
├── tests/                       ← unit + integration tests
├── bench/                       ← reproducible perf harness vs incumbents
└── CMakeLists.txt
```

---

## Documentation

| Doc | Description |
|-----|-------------|
| [Architecture](docs/ARCHITECTURE.md) | Deep design for current rung (Rungs 1-3 today) |
| [Glossary](docs/GLOSSARY.md) | Plain-English definitions of every term used |
| [Research](docs/RESEARCH.md) | Landscape comparison of existing proxies and Switchyard's enhancements |

---

## Roadmap

- [ ] **Rung 1** — `io_uring`-native TCP echo (kqueue fallback for macOS)
- [ ] **Rung 2** — Zero-copy HTTP/1.1 parser + canned response
- [ ] **Rung 3** — Single-backend forwarder with `writev`/`splice`
- [ ] **Rung 4** — Multi-backend round-robin with RCU endpoint table
- [ ] **Rung 5** — Global lockless upstream connection pool
- [ ] **Rung 6** — Active health checks
- [ ] **Rung 7** — Timeouts, retries, retry budgets
- [ ] **Rung 8** — kTLS-first TLS termination
- [ ] **Rung 9** — HTTP/2 with zero-copy frame handling
- [ ] **Rung 10** — RCU-based hot config reload
- [ ] **Rung 11** — Wait-free per-CPU observability
- [ ] **Rung 12** — Reproducible benchmarks vs nginx/HAProxy/Envoy

---

<div align="center">
<b>Built to understand what the abstractions above us hide — and to prove the incumbents leave real performance on the table.</b>
</div>
