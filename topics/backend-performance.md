---
title: Backend Performance Notes
date: 2026-07-21
tags: [backend, performance, runtimes, concurrency, throughput, latency]
source: own knowledge — not verified against primary sources; check official docs and run your own benchmarks before citing specific numbers
---

# Backend Performance Notes

Working notes comparing backend runtimes and the levers that actually move performance. Content is synthesized from general knowledge, not a primary-source research pass — treat every number as order-of-magnitude and benchmark your own workload before deciding. The headline point: past a certain scale, framework choice matters far less than I/O model, database access, and caching.

## Runtime / language comparison

Rough positioning for typical web-service (JSON-over-HTTP, DB-backed) workloads. "Throughput" here means requests/sec on CPU-bound routing + serialization, not real apps hitting a database — where the DB usually dominates.

| Runtime | Concurrency model | Raw throughput | Where it shines | Main cost |
|---|---|---|---|---|
| **Go** | Goroutines (M:N scheduler) | Very high | High-concurrency services, low memory/goroutine, single static binary | More boilerplate, manual error handling |
| **Rust** (Axum, Actix) | async/await on Tokio | Highest | Latency-critical, predictable tail latency, no GC pauses | Steep learning curve, slower to write |
| **Java/JVM** (Spring, Quarkus) | Threads / virtual threads (Loom) | High | Mature ecosystem, virtual threads made blocking code cheap again | JVM warmup, memory footprint (native-image via GraalVM helps) |
| **Node.js / Bun** | Single-threaded event loop | Medium-high | I/O-bound APIs, shared JS with frontend; Bun's runtime is notably faster than Node | CPU-bound work blocks the loop — offload to workers |
| **Python** (FastAPI, async) | async event loop / GIL | Medium | Fast to build, ML/data ecosystem | GIL limits CPU parallelism; scale with multiple processes |

**Rule of thumb**: I/O-bound service that talks to a DB and other services → almost any modern async runtime is fine, pick for ecosystem/team familiarity. CPU-bound or strict tail-latency SLA → Go or Rust. Sharing code/skills with a JS frontend → Node or Bun.

## What actually determines throughput and latency

Ranked by how often it's the real bottleneck — usually well above language choice.

1. **Database access** — the top bottleneck in most services. N+1 queries, missing indexes, and no connection pooling cost more than any runtime difference. Fix the query before switching languages.
2. **I/O model** — blocking vs non-blocking. A single blocking call on an event-loop runtime (Node, async Python) stalls everything. On thread-per-request models (classic JVM, Go) it only ties up one worker.
3. **Serialization** — JSON encode/decode is a real CPU cost at scale. Consider faster codecs (protobuf, msgpack) or streaming for large payloads.
4. **Caching** — the cheapest large win. See below.
5. **Framework overhead** — real but usually the smallest of these. Micro-benchmark rankings (TechEmpower etc.) rarely survive contact with a database-backed workload.

## Concurrency models

The single biggest architectural fork.

### Event loop (Node, Bun, async Python, Nginx)
One thread, non-blocking I/O, callbacks/promises. Excellent for many concurrent I/O-bound connections with low memory. **Any CPU-heavy or accidentally-blocking call freezes all requests** — offload to worker threads/processes.

### Thread-per-request (classic JVM, Go goroutines, Ruby/Puma)
Each request gets its own thread/goroutine; blocking code is fine because only that unit blocks. Simpler mental model. OS threads are heavy (~1 MB stack), so this historically capped concurrency — **goroutines** (cheap, ~KB) and **JVM virtual threads** (Loom) removed that ceiling while keeping blocking-style code.

### Multi-process
Run N processes behind a load balancer to sidestep a single-thread or GIL limit (Python `gunicorn -w`, Node `cluster`/PM2). Scales across cores but each process has its own memory and no shared in-process state — push shared state to Redis/DB.

## Caching layers (in priority order)

1. **Client / CDN** — cache at the edge with proper `Cache-Control` / `ETag`; the request never reaches your backend. Highest ROI for cacheable content.
2. **Application cache** — Redis/Memcached for computed results, session data, hot rows. Watch invalidation and stampede (add jitter/locks on expiry).
3. **In-process / memory cache** — fastest (no network hop) but per-instance and lost on restart; inconsistent across a multi-instance fleet.
4. **Database query cache / materialized views** — precompute expensive aggregations instead of recomputing per request.

## Measuring — don't guess

- **Load test** with `wrk`, `k6`, or `oha`; report **p50 / p95 / p99 latency**, not just average — tail latency is what users feel and what averages hide.
- **Profile** to find the real hotspot (`pprof` for Go, `async-profiler` for JVM, `clinic`/`--prof` for Node, `py-spy` for Python) before optimizing.
- **Watch saturation**: CPU, memory, DB connection pool, and file-descriptor limits. Throughput collapses at the *first* resource to saturate, which is often the connection pool, not the CPU.
- Benchmark the **whole path** (through the DB and dependencies), not a hello-world route — synthetic framework benchmarks rarely predict production.

## Practical approach

Start with the boring wins before rewriting anything: fix N+1 queries and add indexes → add connection pooling → cache at CDN/Redis → only then consider a faster runtime. A profiled, well-indexed Python or Node service beats an unprofiled Rust one. Reach for Go/Rust when you have a *measured* CPU or tail-latency wall that architecture and caching can't move.
