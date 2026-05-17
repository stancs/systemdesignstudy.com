---
title: Scaling Strategies
description: Vertical vs horizontal scaling, stateless services, autoscaling, and how to actually take a system from 1k QPS to 1M QPS without lying about it.
---

"How do you scale this?" is the question that closes most system design interviews. The bad answer is "add more servers." The good answer walks through the bottleneck, picks a strategy that addresses it, and names what the next bottleneck will be.

This page is the vocabulary you need to do that.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 240" role="img" aria-label="Vertical scaling: one bigger box. Horizontal scaling: many boxes." style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Vertical vs horizontal scaling</text>
  <g transform="translate(0,45)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Vertical — one bigger box</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="40" y="20" width="60" height="60" rx="8"/>
      <rect x="180" y="20" width="80" height="100" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="70" y="55">app</text>
      <text x="220" y="55" font-weight="600">app</text>
      <text x="220" y="80" font-size="10">8x CPU, RAM</text>
      <text x="220" y="95" font-size="10">faster disk</text>
    </g>
    <text x="160" y="150" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Zero code change. Hard ceiling.</text>
    <text x="160" y="168" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Still one machine to fail.</text>
  </g>
  <g transform="translate(320,45)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Horizontal — many boxes</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="40" width="50" height="40" rx="6"/>
      <rect x="80" y="40" width="50" height="40" rx="6"/>
      <rect x="140" y="40" width="50" height="40" rx="6"/>
      <rect x="200" y="40" width="50" height="40" rx="6"/>
      <rect x="260" y="40" width="50" height="40" rx="6"/>
      <rect x="50" y="90" width="50" height="40" rx="6"/>
      <rect x="110" y="90" width="50" height="40" rx="6"/>
      <rect x="170" y="90" width="50" height="40" rx="6"/>
      <rect x="230" y="90" width="50" height="40" rx="6"/>
    </g>
    <text x="160" y="150" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Needs stateless or shardable design.</text>
    <text x="160" y="168" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Fault-tolerant by construction.</text>
  </g>
</svg>

## The two axes

**Vertical scaling (scaling up).** Bigger machine. More CPU, more RAM, faster disks, faster network. Pros: zero application changes, instantaneous performance lift, simple operations. Cons: hard ceiling (the biggest cloud instance is finite and very expensive), single point of failure unless paired with replication.

**Horizontal scaling (scaling out).** More machines. Pros: theoretically unlimited capacity, fault-tolerant by construction. Cons: requires the application to be designed for it (stateless or shardable), introduces a coordination layer, multiplies operational cost.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 260" role="img" aria-label="Scaling ladder: get more out of one box, replicate reads, cache, queue, shard" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">The scaling ladder (climb in order)</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="60" y="180" width="100" height="40" rx="8"/>
    <rect x="170" y="150" width="100" height="40" rx="8"/>
    <rect x="280" y="120" width="100" height="40" rx="8"/>
    <rect x="390" y="90" width="100" height="40" rx="8"/>
    <rect x="500" y="60" width="100" height="40" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
  </g>
  <g fill="currentColor" text-anchor="middle" font-size="11">
    <text x="110" y="202" font-weight="600">1. Scale up</text>
    <text x="110" y="216">one bigger box</text>
    <text x="220" y="172" font-weight="600">2. Read replicas</text>
    <text x="220" y="186">followers</text>
    <text x="330" y="142" font-weight="600">3. Cache</text>
    <text x="330" y="156">Redis / CDN</text>
    <text x="440" y="112" font-weight="600">4. Queue</text>
    <text x="440" y="126">async work</text>
    <text x="550" y="82" font-weight="600">5. Shard</text>
    <text x="550" y="96">split writes</text>
  </g>
  <text x="320" y="245" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.7">Reaching step 5 immediately on a 1k-QPS workload is a red flag.</text>
</svg>

The honest sequence for most systems is **vertical first, then horizontal**:

1. Get more out of one box (better algorithms, more cores, more RAM, NVMe).
2. Add read replicas if reads dominate.
3. Add a cache layer.
4. Add a queue to decouple slow work from the request path.
5. Shard the stateful tier when a single leader cannot keep up.

Reaching step 5 immediately, on a 1k QPS workload, is a red flag. So is staying at step 1 when the numbers obviously demand step 5.

## Stateful vs stateless

The single most important architectural decision for scaling is **where state lives**.

**Stateless services** keep no per-user data in memory between requests. Any instance can handle any request. Scaling them is trivial: spin up more, route traffic, done. This is what you should aim for on the application tier.

**Stateful services** keep data in memory or on local disk. Different users (or different keys) live on different instances. Scaling them requires sharding, replication, and careful failover. Databases, caches, and connection-oriented services (WebSockets) are stateful by nature; everything else should be stateless.

The standard pattern: keep all stateful infrastructure behind well-defined services (database, cache, queue) and make every application tier stateless. The instant a senior engineer hears that you've parked session data on the local disk of an app server, they will mark you down.

## Autoscaling

Autoscaling means **automatically adjusting the number of instances** of a stateless tier based on load. Two flavors:

**Reactive autoscaling.** Scale based on observed metrics (CPU, request rate, queue depth). Cheap, easy, works for most workloads. Pitfall: there's a lag between load spike and capacity arriving — usually 30–120 seconds for fresh instances to boot and warm up.

**Predictive autoscaling.** Use historical patterns to scale *before* the load arrives. Useful for predictable diurnal patterns. Most teams don't need it.

Key parameters worth mentioning:

- **Scale-out aggressiveness** — how quickly to add capacity (faster = safer for users, more expensive).
- **Scale-in conservatism** — how slowly to remove capacity (slower = more resilient to short dips, more expensive). Typically scale out fast, scale in slow.
- **Warm-up time** — how long an instance takes to start serving real traffic at full rate.
- **Minimum instance count** — never go below this even during quiet periods.
- **Maximum instance count** — guardrail to limit blast radius of a runaway scale event.

A common mistake: scaling on CPU alone. CPU is downstream of the actual bottleneck for many workloads. **Request queue depth** or **p99 latency** are often better signals.

## Caching and read replicas — the easy wins

Before sharding, exhaust these:

- **Caching** at the right layer. A 95% hit rate cuts origin load 20x. See [Caching](/data/caching/).
- **Read replicas.** A read-heavy database scales beautifully with followers serving reads. Single-leader writes are still capped, but reads scale roughly linearly with replicas.
- **CDN for static and cacheable content.** Frees up your origin entirely for the dynamic path.
- **Connection pooling and HTTP keep-alive.** Doesn't sound like scaling but eliminates the most common low-effort waste at moderate scale.

If your scale problem can be solved by reaching for one of these, **do that first** and say so. *"We're at 100k QPS on reads; adding two read replicas and a Redis cache layer covers us for the next year before we need to talk about sharding"* is a more senior answer than "let's shard."

## When to shard the stateful tier

Reach for [sharding](/data/sharding/) when:

- Write throughput exceeds what a single leader can handle (~50–100k writes/sec on top-tier hardware).
- The dataset has grown beyond what a single host can store.
- A single hot table is dragging the whole database down.

Always pair the sharding decision with a **shard key justification** — the access pattern that makes sharding work for you. "We'll shard by user_id because every query already filters on user_id" is right; "we'll shard because we need scale" is wrong.

## Async, batching, and queues

Sometimes the right answer to "how do we handle 10x traffic?" is not "scale the request path" but **"move the work off the request path."**

- **Async processing.** Accept the request, enqueue the work, return immediately. The user sees fast response times; workers chew through the queue at their own rate. See [Message Queues](/data/message-queues/).
- **Batching.** Combine many small operations into fewer larger ones. A single bulk DB insert can be 10x faster than 1000 individual ones.
- **Pre-computation.** Compute expensive results offline (every minute, every hour) and serve them from a cache. Trending feeds, leaderboards, and aggregations frequently work this way.

Each of these trades some latency or freshness for dramatically lower request-path cost. Always pair them with the freshness window you're accepting.

## Multi-region

The last scaling axis, and the most expensive. Reasons to go multi-region:

- **Latency** — global users want sub-100ms responses.
- **Availability** — a single regional outage shouldn't kill the system.
- **Compliance** — data residency requirements forbid storing certain users' data outside their region.

Three rough shapes:

- **Single writer, geo-replicated reads.** Easy. Reads scale globally; writes still pay one regional round trip. Acceptable for read-heavy systems.
- **Geo-sharded.** Each region owns a subset of users (or tenants). Reads and writes for those users stay local. Cross-region operations are explicit and rare. The most common compromise.
- **Active-active multi-master.** Every region accepts writes for everyone. Best latency, hardest correctness story (conflicts, divergence, reconciliation). Reach for it only when the data model truly supports it (e.g., CRDTs) or when the cost of being wrong is acceptable.

A senior answer to "make it global" is rarely "active-active" — it's usually "geo-shard with regional failover."

## What to say in an interview

A scaling narrative the interviewer wants to hear:

> *"At 10x current load — about 1M QPS read, 30k QPS write — the bottlenecks shift in this order: app tier first (we just autoscale stateless instances), then the database, where the read replicas can't keep up. We add a Redis cache layer first because the read pattern is heavily skewed; if that's not enough, we shard the database by user_id with consistent hashing. Multi-region only if global latency becomes a hard requirement — geo-shard before active-active."*

Three named steps, in order, with the reasoning. That is what scaling looks like as a discipline, not a slogan.
