<div align="center">

<img src="./assets/banner.svg" alt="Switchyard" width="100%"/>

<br/>
<br/>

[![License: MIT](https://img.shields.io/badge/License-MIT-3B82F6?style=for-the-badge)](LICENSE)
[![C++](https://img.shields.io/badge/C%2B%2B-20-00599C?style=for-the-badge&logo=cplusplus&logoColor=white)](https://en.cppreference.com/w/cpp/20)
[![Status](https://img.shields.io/badge/Status-Rung%200%20Planning-lightgrey?style=for-the-badge)](docs/ARCHITECTURE.md)
[![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20macOS-333?style=for-the-badge&logo=linux&logoColor=white)](#)

</div>

---

## The Problem

**2:47 AM. PagerDuty fires. The mobile team pushed a config change to nginx to add a new upstream. Every WebSocket connection in production just dropped.**

The chat app disconnected mid-conversation. Live dashboards went blank. Uploads restarted from 0%. Mobile clients started hammering the reconnect endpoint, burning battery and eating capacity.

The change was one line in `nginx.conf`. The reload was "graceful." But the workers still hard-closed every long-lived connection after the drain timeout — because in nginx's world, a config reload means new worker processes, and the old ones eventually die.

This is what today's reverse proxies (nginx, HAProxy, Envoy) still leave on the table:

| Pain point | What actually breaks | End-user symptom |
|---|---|---|
| Reload drops long-lived connections | New workers spawned; old ones drain and hard-close | WebSocket/SSE/gRPC streams die on every config change |
| Upstream DNS goes stale | Hostnames resolved once at config load | 502 spikes for minutes after every K8s pod reschedule |
| Cross-instance state fragments | Rate limits, sessions, TLS tickets are per-process | Users get 429s they don't deserve; carts randomly reset |
| Observability gaps | Logs say what happened, not why or which phase | "App is slow" incidents last hours instead of minutes |
| Retry storms | Naive retry-on-5xx with no budget | 1% partial outage becomes a 100% total outage |

**The common thread:** every one of these is a failure of *state* — state that's thrown away, goes stale, isn't shared, or isn't observable.

---

## The Thesis

Switchyard treats **state as a first-class primitive.** Config, upstream DNS resolutions, health status, session cache, rate-limit counters, TLS tickets, connection pools — all of them are:

- **Hot-swappable** — updated atomically without dropping in-flight requests or connections
- **Shared** — replicated across proxy instances so users see consistent behavior
- **Introspectable** — every piece of state readable at runtime via an admin API

Everything else (HTTP/1.1 parser, HTTP/2, TLS, LB algorithms, retries) is table stakes. The differentiator is that Switchyard was built with state-consistency as the core design principle, not bolted on after the fact.

---

## Mental Model

A reverse proxy is a mailroom clerk. Clients hand their packages (requests) to the clerk. The clerk reads the label (URL), picks the right desk (backend), delivers it, and brings the reply back. From outside, nobody knows how many desks are on Floor 12 or which one is faster today — that's the clerk's job.

```
              ┌─────────────────────────────────────────────┐
              │              SWITCHYARD                     │
   Clients ──▶│  TCP accept → TLS → HTTP parse → route      │──▶ Backends
              │  → LB pick → pool → forward → response      │
              │                                             │
              │  ── Hot-swappable state layer ──            │
              │  config · DNS · health · sessions · limits  │
              └─────────────────────────────────────────────┘
                                 ▲
                                 │
                         admin API + traces
```

---

## The Build Ladder

Each rung is understandable before starting the next. We plan each rung deeply as we approach it — not months in advance.

| Rung | Milestone | Estimate |
|-----:|-----------|:--------:|
| 1  | TCP echo server — accept, read, write, close | ~1 week |
| 2  | HTTP/1.1 parser + canned response | ~1 week |
| 3  | Single-backend forwarder (it's a proxy!) | ~1 week |
| 4  | Multi-backend + round-robin load balancing | ~1 week |
| 5  | Upstream connection pooling | ~1 week |
| 6  | Health checks | ~1 week |
| 7  | Timeouts and retries with budget | ~2 weeks |
| 8  | TLS termination (BoringSSL/OpenSSL) | ~2 weeks |
| 9  | HTTP/2 support | ~3-4 weeks |
| 10 | Hot config reload — state-first thesis begins | ~3 weeks |
| 11 | Observability — per-request traces + admin API | ~2 weeks |
| 12 | Shared cross-instance state (replication) | ~4+ weeks |

**Rungs 1-3** = a terrible-but-working reverse proxy. ~3 weeks.
**Rungs 1-7** = something you could put in front of a hobby app. ~2-3 months.
**Rungs 1-12** = the full state-first vision. 6-12 months of focused work.

---

## Why C++

- Zero-cost abstractions and manual memory control matter on the hot path of a network proxy.
- Direct access to OS primitives (epoll, `io_uring`, kqueue) with no runtime in the way.
- Mature crypto/TLS ecosystem (OpenSSL, BoringSSL) with native bindings.
- Forces explicit reasoning about ownership, lifetimes, and concurrency — the exact skills backend fundamentals demand.

Go and Rust are both reasonable alternatives. C++ is chosen deliberately for the learning value: the goal is to understand backend fundamentals, and C++ leaves nothing hidden.

---

## Repo Layout (planned)

```
switchyard/
├── README.md                    ← you are here
├── LICENSE
├── docs/
│   ├── ARCHITECTURE.md          ← deep plan for the current rung
│   ├── GLOSSARY.md              ← plain-English definitions
│   └── decisions/               ← ADRs for non-obvious choices (added as we go)
├── src/                         ← C++ source (added at Rung 1)
├── tests/                       ← unit + integration tests (added at Rung 1)
└── CMakeLists.txt               ← build system (added at Rung 1)
```

---

## Documentation

| Doc | Description |
|-----|-------------|
| [Architecture](docs/ARCHITECTURE.md) | Deep design for the current rung (Rungs 1-3 today) |
| [Glossary](docs/GLOSSARY.md) | Plain-English definitions of every term used in the project |

---

## Roadmap

- [ ] **Rung 1** — TCP echo server (accept loop, read, write, close)
- [ ] **Rung 2** — HTTP/1.1 request parser + canned response
- [ ] **Rung 3** — Single-backend forwarder (working reverse proxy)
- [ ] **Rung 4** — Multi-backend round-robin load balancing
- [ ] **Rung 5** — Upstream connection pooling
- [ ] **Rung 6** — Active health checks
- [ ] **Rung 7** — Timeouts, retries, retry budgets
- [ ] **Rung 8** — TLS termination
- [ ] **Rung 9** — HTTP/2 support
- [ ] **Rung 10** — Hot config reload (state-first begins)
- [ ] **Rung 11** — Structured per-request observability + admin API
- [ ] **Rung 12** — Cross-instance shared state (replication)

---

<div align="center">
<b>Built to understand what the abstractions above us hide.</b>
</div>
