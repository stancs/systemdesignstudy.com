---
title: Caching Strategies
description: Cache topologies, read and write patterns (cache-aside, read-through, write-through, write-back), eviction policies, and the failure modes every caching design has to handle.
---

A cache is a smaller, faster store that sits in front of a slower, larger one. Caches make slow systems feel fast and large systems feel cheap. They also introduce a second copy of your data, which means a second source of bugs. Designing the cache well is one of the most common deep-dive topics in system design interviews.

<img src="/diagrams/data/caching-1.svg" alt="The chain of caches between user and origin" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

## Where caches live

Caches show up at every layer of a system. In rough order from client to server:

- **Browser / mobile app cache.** Closest to the user, fastest, completely free for the server. Governed by HTTP cache headers.
- **CDN cache.** Edge POPs around the world. See [CDN](/networking/cdn/).
- **Reverse proxy / API gateway cache.** Nginx, Varnish, Envoy. Caches at the entrance to your cluster.
- **Application-level cache.** In-process (Caffeine, Guava). Microseconds, but per-instance.
- **Distributed cache.** Redis, Memcached. Shared across all app servers in the cluster.
- **Database query cache or buffer pool.** The database's own memory of recently-accessed pages.

A single read may pass through several of these. Each level is bigger and slower than the one before it. In an interview, when you say "we cache," specify *which* cache you mean.

## The four caching patterns

When the app server needs data, four patterns describe how the cache interacts with the underlying store.

<img src="/diagrams/data/caching-2.svg" alt="Cache-aside flow: app checks cache, falls through to database on miss, then populates the cache" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

### Cache-aside (lazy loading)

The app talks to the cache. On miss, it reads from the database and populates the cache. The cache itself doesn't know about the database.

```
1. App reads cache.
2. If hit, return.
3. If miss, app reads DB, writes the value to cache, returns it.
```

- **Pros:** Simple. Only cached data lives in the cache (memory-efficient). The cache surviving a database outage is straightforward.
- **Cons:** Initial requests are slow (cache misses). Stale data is the app's problem.

This is the default for most production systems.

### Read-through

The app talks only to the cache. The cache itself is responsible for loading from the database on miss.

- **Pros:** Cleaner app code; cache is the single source of read truth.
- **Cons:** Requires a cache that supports the pattern (e.g., a library or a CDN with origin pull).

### Write-through

Writes go to the cache, which synchronously writes to the database before returning.

- **Pros:** Cache and database stay in sync; reads are always served from a fresh cache.
- **Cons:** Writes pay both cache and DB latency. Every write populates the cache even if it's never read.

### Write-back (write-behind)

Writes go to the cache and return immediately. The cache flushes to the database asynchronously.

- **Pros:** Lowest write latency.
- **Cons:** If the cache dies before flushing, you lose writes. Used carefully in narrow contexts; rarely the right answer for durable data.

In real systems you almost always combine **cache-aside for reads** with **write-through (or invalidate-on-write)** for writes. That combo handles the common cases without the durability risk of write-back.

## Invalidation: the actually hard part

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

Three common strategies, in increasing complexity:

**TTL-based.** Each cache entry has an expiry; readers tolerate up to that much staleness. Simple, robust, easy to reason about. The trade-off: short TTLs reduce staleness but increase miss rate and load on the origin.

**Explicit invalidation on write.** When the app updates the underlying store, it also deletes the cache entry. Fresh on the next read.

**Update on write.** Like above, but writes the new value into the cache instead of just deleting it. Faster subsequent reads, but you have to be sure the value you're writing is correct (race conditions live here — two writers can race and the loser's value sticks).

Most senior designs use **TTL as a backstop** plus **explicit invalidation on write** for known update paths. The TTL bounds how stale anything can ever be; the explicit invalidation keeps the common case fresh.

## Eviction policies

When the cache fills up, something has to leave. Common policies:

- **LRU (Least Recently Used).** Evict the entry not accessed for the longest time. Most common default.
- **LFU (Least Frequently Used).** Evict the entry with the fewest accesses. Better when access frequency, not recency, predicts future use.
- **FIFO.** Evict the oldest entry by insertion order. Simple, rarely optimal.
- **TTL-driven.** Entries leave when their TTL expires regardless of access.
- **Random.** Surprisingly close to LRU at much lower overhead, used in some constrained environments.

For most workloads, LRU is fine. Workloads with strong frequency skew (a few extremely popular keys) benefit from LFU or hybrid policies (W-TinyLFU, used in Caffeine).

## Sizing and hit rate

The single most important cache metric is **hit rate** — fraction of requests served from cache. A cache with a 99% hit rate carries 100x the effective load of the underlying store; a cache at 50% hit rate is doing roughly half the work and pays half the cost.

Two rough heuristics:

- Hit rate has diminishing returns as you grow the cache. Going from 90% to 99% may require 10x the cache size.
- The working set follows Pareto-like distribution: ~80% of accesses go to ~20% of keys. That's why even small caches are dramatically useful.

Always say what you expect the hit rate to be and what happens at miss — your origin sees the full miss traffic, and you need to be sure it can.

## Failure modes you must mention

<img src="/diagrams/data/caching-3.svg" alt="Thundering herd on cache expiry, before and after request coalescing" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

**Cache stampede / thundering herd.** A hot key expires and a thousand concurrent requests all miss simultaneously. They all hit the origin, which falls over. Mitigations: **request coalescing** (one origin fetch per unique key, others wait), **probabilistic early expiration** (refresh slightly before TTL), or **never-expire + background refresh** for the hottest keys.

**Cold cache after restart.** A freshly started cache has zero hits. Mitigations: warm the cache from a snapshot, drain traffic gradually, or accept the warming period and capacity-plan the origin to handle it.

**Inconsistency between cache and store.** A write succeeds in the DB but the cache invalidation fails (network blip). Stale reads continue forever — or until the TTL saves you. This is exactly why TTL is your backstop.

**Hot key.** One key gets 100x more traffic than the rest. The shard holding it saturates. Mitigations: local in-process caching of hot keys above the distributed cache; per-key replication; manual splitting (`user:42:a`, `user:42:b`).

**Cache-as-source-of-truth.** Tempting and wrong. Caches lose data. Always treat the cache as a performance layer; the database is the truth.

## What to say in an interview

A clean, defensible caching paragraph:

> *"The hot read path is cache-aside against Redis, keyed by `user:{id}`, with a 60-second TTL. Writes invalidate the cache before returning. The cache is sized to hold the working set — about 20% of users at any given time — which our load test puts at a ~95% hit rate. To handle hot keys for popular users we add a small in-process cache on the app servers in front of Redis. The thundering-herd risk is mitigated by request coalescing on misses."*

Five concrete decisions, each tied to a reason. That is the deep dive interviewers want to hear.
