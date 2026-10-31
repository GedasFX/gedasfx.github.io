---
layout: post
title: "Node, Python, .NET and .NET AOT on a small CRUD API: day one vs tuned"
date: 2026-09-27
categories: [benchmark]
tags: [dotnet, aot, node, python, sqlite, postgres, performance]
---

In my [last post](/dotnet-aot-benchmark/) I measured .NET native AOT against my Node, Python and regular .NET templates. The catch was that every app only answered `/health`. That shows what the runtime costs, but not what happens when the app does real work.

So this time each stack got a small CRUD API over SQLite, built twice: once with raw SQL and once with an ORM. I measured each service as first written (something you would get from following a getting started guide), then again after a few hours of tuning.

In short, without any magic tuning, .NET is the safe pick, and Python is a big trap. After tuning, Node, .NET and AOT end up close together.

# The setup

The API has four endpoints over two tables, products and categories:

| Endpoint | Does |
|---|---|
| `GET /products/{id}` | the product joined to its category, or 404 |
| `POST /products` | validates the body, inserts, returns the product with its category; 409 on a duplicate SKU |
| `DELETE /products/{id}` | 204, or 404 |
| `GET /health` | `ok` |

The database starts with 50 categories and 10,000 products. The ids have gaps, so a random id finds a product 80% of the time.

Eight services, two per stack:

| Stack | Raw SQL | ORM |
|---|---|---|
| Node 24 | `node:sqlite` | Drizzle |
| Python 3.14 (FastAPI) | `sqlite3` | SQLAlchemy |
| .NET 10 | Dapper | EF Core |
| .NET 10 AOT | Dapper.AOT | EF Core |

**I didn't write these services by hand.** Each one was built by a Claude agent from the same written spec, starting from my template for that stack. A service counted as done when the template's own checks passed (lint, format, types, tests) and an 18-check API contract passed against the Docker image. Then a second round of agents tuned them, with about 3 hours per stack. That's how 28 services got built and measured. It also means "how easy was it" partly measures how well the model knows each stack.

The load is [k6](https://k6.io/) with 50 virtual users: 75% reads, 20% creates and 5% deletes, 10 s of warm-up and then 30 s measured. Every service ran alone under `--cpus 2 --memory 512m`, just like last time. Every number below is the mean of two clean rounds (consistent - within 2%).

# Day one: my quickstart templates, no tuning

| | Node | Python | .NET | .NET AOT |
|---|---|---|---|---|
| Raw SQL (req/s) | 15.8k | 0.7k | **20.6k** | **20.6k** |
| ORM (req/s) | 3.5k | 0.7k | 8.0k | build fails (6.0k with warnings allowed) |
| Idle / peak memory, raw | 29 / 84 MiB | 50 / 110 MiB | 22 / 100 MiB | **11 / 68 MiB** |
| Image, disk / compressed | 330 / 80 MB | 222 / 52 MB | 184 / 56 MB | **38 / 11 MB** |
| Clean build | 10–14 s | 11–14 s | 10–12 s | 36–39 s |
| Startup, incl. `docker run` (~300 ms) | 358 ms | 1.2–1.7 s | 500 ms | **314 ms** |

**.NET is fast out of the box.** JIT and AOT tie at 20.6k with plain Dapper, and neither needed changes to hold up under load. The cost is code: my .NET template is layered, so the service came out at ~270 lines over 10–11 files. The others needed 150–200 lines over 3–4 files.

**Python was suspiciously slow.** The template uses sync FastAPI endpoints. Each request makes several `sqlite3` calls from a 40-thread pool, and the threads queue for the GIL (p50 68 ms). The raw SQL one also hid a concurrency bug that only showed under load: `sqlite3` without `check_same_thread=False` failed 347 of 400 concurrent requests. The ORM one had that setting already, straight from FastAPI's docs.

**EF Core doesn't work with AOT.** It fails the build on AOT warnings (IL2026, IL3050). With warnings allowed it builds, but it needed four workarounds and reached only 6.0k. Dapper.AOT works fine, as in the last post.

## Postgres check

Looking at the results, I wondered if the bottleneck was the database choice. I tried Postgres and found it made things *slower*, even with Postgres on its own 2 CPUs: 2.4–2.9x for .NET and AOT raw, 1.3–1.7x for the ORMs and Node. At 10,000 rows, SQLite isn't the bottleneck. At a bigger scale it could be, but I'm testing the API, not the database, so SQLite stays.

# Tuned: surprisingly close results

| | Node | Python | .NET | .NET AOT |
|---|---|---|---|---|
| Raw SQL (req/s, p99) | 37.4k, 2.8 ms | 14.4k, 6.7 ms | **44.2k, 2.0 ms** | 39.7k, 2.1 ms |
| ORM (req/s) | 36.7k | 5.2k | 25.0k | 13.3k (warnings allowed) |
| Raw + in-memory cache (req/s) | 31.5k | 11.8k | **51.8k** | 48.8k |

What made the difference, biggest first:

| Stack | Changes |
|---|---|
| Node | writes serialised in-process (SQLite's busy-wait froze the event loop); uWebSockets.js instead of `node:http` (+44% per thread); 2 `worker_threads` on one port |
| Python | `async def` handlers (×4.8); one connection per process (×2.3); 2 uvicorn workers; Litestar + msgspec instead of FastAPI + Pydantic (+14%) |
| .NET | connections opened once with prepared commands, without Dapper, which re-prepares (20.4k → 33.7k); thread pool capped at the CPU count; group commit (+4–9%) |
| .NET AOT | the same three: prepared statements (+47%), group commit (+15–25%), thread-pool cap |

**If there is one finding to take away here, it's to limit the thread pool.** Under the test, the EF service was throttled in 187 of 251 periods: p50 5 ms, p99 43 ms. One line fixed it: `ThreadPool.SetMaxThreads(Environment.ProcessorCount, …)`. .NET did see its 2 CPUs (that line caps the pool at exactly 2), so my best guess is that blocking database calls made the pool add threads beyond that, and the extra threads used up the CPU quota early in each 100 ms period. I've only seen this under Docker; I haven't checked k8s.

**Tuning Node ORM was a pleasant surprise:** prepared Drizzle queries reach 98% of raw SQL. That is exceptionally good compared to the others: EF Core stayed 1.8x behind raw, and SQLAlchemy 2.8x.

**AOT ends 10% behind JIT** on the same design (39.7k against 44.2k). It caught me off guard for sure, but that just shows how optimized the JIT is (or how much room for improvement is left for AOT).

## Cache test

It is not really fair to add a cache on top for this test, but I really wanted to see just how much a cache does. And to be frank, for this test - surprisingly little. +17% for .NET and +23% for AOT. Primary-key reads are just that cheap!

On Node and Python the cached version had to run as one process, since a per-process cache goes stale across processes, but then you lose the benefit of running them in parallel, and that's its own can of worms. Maybe a good topic to recheck later.

# Conclusions I concluded (What I'd pick)

- **.NET** when I want it to just work: fastest on day one, typed, and the ORM works. It needs the thread-pool cap in containers, but overall a good choice.
- **Node** for web apps. It's surprisingly quick: not perfect, but perfectly serviceable for all web app needs. And with things like Svelte and Next.js you get to do the UI on top, which is a nice bonus. For enterprise size apps maybe I would look for a better backend for API, but my projects are definitely not there yet.
- **.NET AOT**: I'll need to rethink this one. Maybe some CLI tool (maybe future comparison to Rust or Go?), basically something that needs to spin up fast. As it stands, not that good a deal, even with the memory benefits.
- **Python** really disappointed me. It took quite a bit of finagling just to get near the other stacks. And frankly, I'm not a fan of the language itself. I'd likely use it only where its rich ecosystem helps, and even then as a sidecar to something else.

# Test setup

- **Host:** WSL2, 16 vCPU, 15 GB RAM, Docker Desktop 29.6.1. Date: 2026-09-27.
- **Images:** `node:24-slim`, `python:3.14-slim`, .NET 10 on `aspnet:10.0-noble-chiseled`, AOT built on `sdk:10.0-noble-aot` into `runtime-deps:10.0-noble-chiseled`.
- **Libraries:** Drizzle 0.45.3, better-sqlite3 13.0.3, SQLAlchemy 2.1.1, EF Core 10.0.12, Dapper 2.1.89, Dapper.AOT 1.1.0, Npgsql 10.0.3, k6 2.3.0, `postgres:18`.
- **Each run:** a fresh copy of the seeded database; startup is the median of 3, from `docker run -d` to the first 200 on `/health`; idle memory is `docker stats` 10 s later; then the contract, which must pass; then the k6 load, with memory sampled every 100 ms from the cgroup.
- **Postgres:** a fresh `postgres:18` per run, with its own `--cpus 2 --memory 1g` and `synchronous_commit=off`, to match SQLite's `synchronous=NORMAL`.
- **k6 isn't the limit:** it pushed 95.9k req/s on `/health` against the AOT service.

# Caveats

- **One machine, one data size, one workload.** The data fits in memory. A larger or more read-heavy dataset would shift the SQLite vs Postgres result.
- **The Postgres services are day-one code on a default Postgres.** They show what switching costs, not how fast Postgres can go.
- **Agents built and tuned everything.** Tuning time was about the same per stack, and each stack might gain more with more time. The tuned Node design (uWebSockets.js, worker threads) is the furthest from its template.
- **Memory includes SQLite's page cache,** which grows during a run.
