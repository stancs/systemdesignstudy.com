---
title: Caching Strategies
description: Cache topologies, read and write patterns (cache-aside, read-through, write-through, write-back), eviction policies, and the failure modes every caching design has to handle.
---

A cache is a smaller, faster store that sits in front of a slower, larger one. Caches make slow systems feel fast and large systems feel cheap. They also introduce a second copy of your data, which means a second source of bugs. Designing the cache well is one of the most common deep-dive topics in system design interviews.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 240" role="img" aria-label="The chain of caches between user and origin" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:12px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Caches at every layer (close to far)</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="10" y="60" width="90" height="50" rx="8"/>
    <rect x="115" y="60" width="90" height="50" rx="8"/>
    <rect x="220" y="60" width="90" height="50" rx="8"/>
    <rect x="325" y="60" width="90" height="50" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="430" y="60" width="100" height="50" rx="8"/>
    <rect x="545" y="60" width="85" height="50" rx="8"/>
  </g>
  <g fill="currentColor" text-anchor="middle">
    <text x="55" y="78" font-weight="600">Browser</text>
    <text x="55" y="96" font-size="10">µs</text>
    <text x="160" y="78" font-weight="600">CDN edge</text>
    <text x="160" y="96" font-size="10">~30 ms</text>
    <text x="265" y="78" font-weight="600">Reverse</text>
    <text x="265" y="96" font-size="10">proxy</text>
    <text x="370" y="78" font-weight="600">App-local</text>
    <text x="370" y="96" font-size="10">µs (RAM)</text>
    <text x="480" y="78" font-weight="600">Redis /</text>
    <text x="480" y="96" font-size="10">Memcached</text>
    <text x="587" y="78" font-weight="600">DB cache</text>
    <text x="587" y="96" font-size="10">buffer pool</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none">
    <path d="M100 85 H115"/>
    <path d="M205 85 H220"/>
    <path d="M310 85 H325"/>
    <path d="M415 85 H430"/>
    <path d="M530 85 H545"/>
  </g>
  <g fill="currentColor">
    <polygon points="111,83 117,85 111,87"/>
    <polygon points="216,83 222,85 216,87"/>
    <polygon points="321,83 327,85 321,87"/>
    <polygon points="426,83 432,85 426,87"/>
    <polygon points="541,83 547,85 541,87"/>
  </g>
  <text x="320" y="170" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">Each level is bigger and slower than the one before it.</text>
  <text x="320" y="190" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">"We cache" only means something when you say <tspan font-style="italic">which</tspan> cache.</text>
</svg>

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

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 280" role="img" aria-label="Cache-aside flow: app reads cache, falls through to database on miss" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Cache-aside (the most common pattern)</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="40" y="100" width="120" height="50" rx="8"/>
    <rect x="260" y="60" width="120" height="50" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="260" y="160" width="120" height="50" rx="8"/>
    <rect x="480" y="160" width="120" height="50" rx="8"/>
  </g>
  <g fill="currentColor" text-anchor="middle">
    <text x="100" y="122" font-weight="600">App</text>
    <text x="100" y="140" font-size="11">read user 42</text>
    <text x="320" y="82" font-weight="600">Cache</text>
    <text x="320" y="100" font-size="11">Redis</text>
    <text x="320" y="182" font-weight="600">Cache miss?</text>
    <text x="320" y="200" font-size="11">fetch + populate</text>
    <text x="540" y="182" font-weight="600">Database</text>
    <text x="540" y="200" font-size="11">Postgres</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none">
    <path d="M160 115 H260 V85"/>
    <path d="M260 100 H160 V125"/>
    <path d="M160 135 H260 V170"/>
    <path d="M260 185 H160 V125"/>
    <path d="M380 185 H480"/>
    <path d="M480 185 H380"/>
  </g>
  <g fill="currentColor">
    <polygon points="256,83 262,85 256,87"/>
    <polygon points="164,123 158,125 164,127"/>
    <polygon points="256,168 262,170 256,172"/>
    <polygon points="164,123 158,125 164,127"/>
    <polygon points="476,183 482,185 476,187"/>
    <polygon points="384,183 378,185 384,187"/>
  </g>
  <g fill="currentColor" font-size="11" opacity="0.85">
    <text x="180" y="80">1. check cache</text>
    <text x="180" y="100">2. if hit, return</text>
    <text x="180" y="170">3. else miss</text>
    <text x="395" y="170">4. read DB</text>
    <text x="395" y="200">5. write cache</text>
  </g>
</svg>

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

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 220" role="img" aria-label="Thundering herd on cache expiry, before and after request coalescing" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Thundering herd, with and without coalescing</text>
  <g transform="translate(0,45)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">No coalescing</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="20" width="60" height="30" rx="6"/>
      <rect x="20" y="55" width="60" height="30" rx="6"/>
      <rect x="20" y="90" width="60" height="30" rx="6"/>
      <rect x="20" y="125" width="60" height="30" rx="6"/>
      <rect x="240" y="70" width="70" height="30" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="50" y="40">req</text>
      <text x="50" y="75">req</text>
      <text x="50" y="110">req</text>
      <text x="50" y="145">req</text>
      <text x="275" y="90" font-weight="600">Origin</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M80 35 H240"/>
      <path d="M80 70 H240"/>
      <path d="M80 105 H240"/>
      <path d="M80 140 H240"/>
    </g>
    <text x="160" y="180" text-anchor="middle" fill="var(--sl-color-accent,#3b82f6)" font-size="11">All N misses hit origin simultaneously.</text>
  </g>
  <g transform="translate(320,45)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">With coalescing</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="20" width="60" height="30" rx="6"/>
      <rect x="20" y="55" width="60" height="30" rx="6"/>
      <rect x="20" y="90" width="60" height="30" rx="6"/>
      <rect x="20" y="125" width="60" height="30" rx="6"/>
      <rect x="110" y="70" width="60" height="30" rx="6"/>
      <rect x="240" y="70" width="70" height="30" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="50" y="40">req</text>
      <text x="50" y="75">req</text>
      <text x="50" y="110">req</text>
      <text x="50" y="145">req</text>
      <text x="140" y="90" font-weight="600">Cache</text>
      <text x="275" y="90" font-weight="600">Origin</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M80 35 H110 V70"/>
      <path d="M80 70 H110"/>
      <path d="M80 105 H110 V100"/>
      <path d="M80 140 H110 V100"/>
      <path d="M170 85 H240"/>
    </g>
    <text x="160" y="180" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">One origin fetch — others wait for it.</text>
  </g>
</svg>

**Cache stampede / thundering herd.** A hot key expires and a thousand concurrent requests all miss simultaneously. They all hit the origin, which falls over. Mitigations: **request coalescing** (one origin fetch per unique key, others wait), **probabilistic early expiration** (refresh slightly before TTL), or **never-expire + background refresh** for the hottest keys.

**Cold cache after restart.** A freshly started cache has zero hits. Mitigations: warm the cache from a snapshot, drain traffic gradually, or accept the warming period and capacity-plan the origin to handle it.

**Inconsistency between cache and store.** A write succeeds in the DB but the cache invalidation fails (network blip). Stale reads continue forever — or until the TTL saves you. This is exactly why TTL is your backstop.

**Hot key.** One key gets 100x more traffic than the rest. The shard holding it saturates. Mitigations: local in-process caching of hot keys above the distributed cache; per-key replication; manual splitting (`user:42:a`, `user:42:b`).

**Cache-as-source-of-truth.** Tempting and wrong. Caches lose data. Always treat the cache as a performance layer; the database is the truth.

## What to say in an interview

A clean, defensible caching paragraph:

> *"The hot read path is cache-aside against Redis, keyed by `user:{id}`, with a 60-second TTL. Writes invalidate the cache before returning. The cache is sized to hold the working set — about 20% of users at any given time — which our load test puts at a ~95% hit rate. To handle hot keys for popular users we add a small in-process cache on the app servers in front of Redis. The thundering-herd risk is mitigated by request coalescing on misses."*

Five concrete decisions, each tied to a reason. That is the deep dive interviewers want to hear.
