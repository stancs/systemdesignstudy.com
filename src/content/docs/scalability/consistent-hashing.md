---
title: Consistent Hashing
description: Why naive hashing collapses under resharding, how consistent hashing fixes it, virtual nodes, and where it shows up in real systems.
---

Consistent hashing is the algorithm behind almost every horizontally scaled cache and distributed database in production. It is also one of the most reliably asked deep-dive topics in system design interviews, because it shows whether you understand the failure mode of the *naive* solution.

## The problem with `hash(key) mod N`

Suppose you want to spread keys across N cache servers. The obvious answer:

```
server = hash(key) mod N
```

With N = 4 servers, every key lands on a deterministic one. Lovely.

Now add a fifth server. The mod changes from 4 to 5, and *almost every key* changes its destination. ~80% of the cache moves, and the new server can't be useful until those keys re-migrate.

Remove a server (one dies). Same disaster in reverse: ~80% of keys move, the surviving servers all suddenly get queried for keys they don't have, and the origin gets blasted while the cache rebuilds.

This is the entire reason consistent hashing exists.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 315" role="img" aria-label="The consistent hash ring with three servers and several keys" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">The hash ring</text>

  <!-- Region arcs: each server owns the arc ending at its dot (clockwise) -->
  <!-- Server A region: C(150°)→A(270°) CW = left/upper-left arc -->
  <path d="M225,225 A110,110 0 0,1 320,60"  stroke="#6090E0" stroke-width="10" fill="none" opacity="0.28"/>
  <!-- Server B region: A(270°)→B(30°) CW = upper-right arc -->
  <path d="M320,60  A110,110 0 0,1 415,225" stroke="#4FAD72" stroke-width="10" fill="none" opacity="0.28"/>
  <!-- Server C region: B(30°)→C(150°) CW = lower arc -->
  <path d="M415,225 A110,110 0 0,1 225,225" stroke="#E08840" stroke-width="10" fill="none" opacity="0.28"/>

  <!-- Ring outline -->
  <circle cx="320" cy="170" r="110" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.45"/>

  <!-- Server nodes -->
  <circle cx="320" cy="60"  r="11" fill="#6090E0"/>
  <circle cx="415" cy="225" r="11" fill="#4FAD72"/>
  <circle cx="225" cy="225" r="11" fill="#E08840"/>

  <!-- Server labels -->
  <g fill="currentColor" font-size="12" font-weight="600">
    <text x="320" y="44"  text-anchor="middle">Server A</text>
    <text x="430" y="229" text-anchor="start">Server B</text>
    <text x="210" y="229" text-anchor="end">Server C</text>
  </g>

  <!-- Key dots on ring — color = assigned server -->
  <!-- Server B keys: k1(-65°), k2(-40°), k3(-10°) on A→B upper-right arc -->
  <circle cx="367" cy="70"  r="5" fill="#4FAD72"/>
  <circle cx="404" cy="99"  r="5" fill="#4FAD72"/>
  <circle cx="428" cy="151" r="5" fill="#4FAD72"/>
  <!-- Server C keys: k4(55°), k5(105°) on B→C lower arc -->
  <circle cx="383" cy="260" r="5" fill="#E08840"/>
  <circle cx="291" cy="276" r="5" fill="#E08840"/>
  <!-- Server A keys: k6(175°), k7(220°) on C→A left arc -->
  <circle cx="210" cy="180" r="5" fill="#6090E0"/>
  <circle cx="236" cy="99"  r="5" fill="#6090E0"/>

  <!-- Key labels -->
  <g font-size="11" fill="currentColor">
    <text x="374" y="54"  text-anchor="start">k1</text>
    <text x="417" y="89"  text-anchor="start">k2</text>
    <text x="442" y="148" text-anchor="start">k3</text>
    <text x="391" y="273" text-anchor="start">k4</text>
    <text x="285" y="291" text-anchor="middle">k5</text>
    <text x="195" y="181" text-anchor="end">k6</text>
    <text x="224" y="89"  text-anchor="end">k7</text>
  </g>

  <text x="320" y="308" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.7">A key belongs to the first server clockwise from its hash position. Key color = assigned server.</text>
</svg>

## The core idea

Imagine a circle (the **hash ring**) representing all possible hash values, from 0 to 2³² − 1 wrapped around. To place items on the ring:

1. Hash each **server** to a point on the ring (`hash(server_id)`).
2. Hash each **key** to a point on the ring (`hash(key)`).
3. A key belongs to the **first server you reach going clockwise** from its hash point.

Adding a server: hash the new server, place it on the ring, and only the keys between its position and the previous (counter-clockwise) server's position move. On a ring of N servers, you move roughly **1/N of the keys** instead of all of them.

Removing a server: its slice of keys gets absorbed by the next server clockwise. Same 1/N magnitude.

That is the entire algorithm. The properties — monotonicity (existing keys mostly stay put), balance (load roughly even), spread (limited duplication), load (each server gets ~1/N of keys) — fall out of this geometry.

## Virtual nodes

Plain consistent hashing has a real problem: with only N points on the ring, the slices between them are uneven. One server might own 30% of the ring while another owns 5%, purely by where their hashes happened to land. Removing a server hands its whole slice to one neighbor, also unevenly.

The fix is **virtual nodes** (vnodes). Each physical server is hashed to many points on the ring — typically 100 to 200 per server. The ring now has thousands of small slices, and load is averaged across many regions. When a server is added, it picks up small slices from many neighbors; when one is removed, its slices spread across many neighbors.

Two other useful properties of vnodes:

- **Heterogeneous capacity.** Give a 2x-bigger server twice as many vnodes. It gets roughly twice the traffic.
- **Smooth rebalancing.** Rather than one disruptive move, vnodes spread the rebalance across many small key ranges in parallel.

Almost every real-world consistent-hash implementation (Cassandra, DynamoDB, Memcached client libraries, Envoy's `ring_hash`) uses virtual nodes.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 320" role="img" aria-label="Hash ring without and with virtual nodes" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Without vs with virtual nodes</text>
  <g transform="translate(160,170)">
    <text x="0" y="-115" text-anchor="middle" fill="currentColor" font-weight="600">Plain ring</text>
    <circle r="80" fill="none" stroke="currentColor" stroke-width="2"/>
    <g fill="var(--sl-color-accent,#3b82f6)">
      <circle cx="0" cy="-80" r="8"/>
      <circle cx="65" cy="50" r="8"/>
      <circle cx="-65" cy="50" r="8"/>
    </g>
    <text x="0" y="120" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Slices are uneven by luck.</text>
    <text x="0" y="136" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Removing a server gives a big slice to its neighbor.</text>
  </g>
  <g transform="translate(480,170)">
    <text x="0" y="-115" text-anchor="middle" fill="currentColor" font-weight="600">With ~12 vnodes each</text>
    <circle r="80" fill="none" stroke="currentColor" stroke-width="2"/>
    <g>
      <circle cx="0" cy="-80" r="5" fill="var(--sl-color-accent,#3b82f6)"/>
      <circle cx="35" cy="-72" r="5" fill="currentColor"/>
      <circle cx="62" cy="-50" r="5" fill="var(--sl-color-accent,#3b82f6)"/>
      <circle cx="79" cy="-15" r="5" fill="currentColor"/>
      <circle cx="78" cy="20" r="5" fill="var(--sl-color-accent,#3b82f6)"/>
      <circle cx="62" cy="50" r="5" fill="currentColor"/>
      <circle cx="30" cy="74" r="5" fill="var(--sl-color-accent,#3b82f6)"/>
      <circle cx="-10" cy="79" r="5" fill="currentColor"/>
      <circle cx="-45" cy="65" r="5" fill="var(--sl-color-accent,#3b82f6)"/>
      <circle cx="-72" cy="35" r="5" fill="currentColor"/>
      <circle cx="-80" cy="0" r="5" fill="var(--sl-color-accent,#3b82f6)"/>
      <circle cx="-65" cy="-45" r="5" fill="currentColor"/>
      <circle cx="-35" cy="-72" r="5" fill="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <text x="0" y="120" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Load smooths to ~1/N.</text>
    <text x="0" y="136" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Removing a server distributes work across many neighbors.</text>
  </g>
</svg>

## Where you find it in real systems

- **Distributed caches.** Memcached has no native distribution — the *client* uses consistent hashing to pick a server. Add a cache node, only 1/N of the keys miss while the cache warms.
- **Dynamo-style stores.** Cassandra and DynamoDB partition data by consistent hashing of the partition key. Replication factor N? Walk N steps clockwise and store on each.
- **Load balancers.** Envoy and HAProxy support `ring_hash` and `maglev` load balancing for sticky routing without state on the LB itself.
- **Sharded application services.** Routing a user_id to the right service instance for in-memory state (a WebSocket connection, a session).

If you mention consistent hashing in an interview, naming at least one of these is good signal.

## Variants worth knowing

**Maglev hashing (Google).** A different approach that achieves similar properties using a lookup table instead of a ring. Slightly less ideal monotonicity but excellent lookup performance and very even distribution. Used in Google's network load balancer and Envoy.

**Jump consistent hash (Google).** A tiny, branch-free function that maps a key to a bucket in 1..N. Excellent load balance and no memory overhead — but you can only add buckets at the end, not remove arbitrary ones. Good fit when buckets only grow.

**Rendezvous (highest random weight) hashing.** For each key, compute `hash(server_id, key)` for every server and pick the highest. Equivalent guarantees to consistent hashing, no ring data structure, but O(N) per lookup — fine for small N (tens of servers).

In an interview, plain consistent hashing with virtual nodes is the right default to describe. Mention maglev or rendezvous only if you have a reason.

## Failure modes and pitfalls

**Skewed keys.** Consistent hashing distributes the *keyspace* evenly, not the *load*. One celebrity user with 10M requests per second still lives on one shard. Mitigate with key splitting (suffix the key with a random component on hot keys, aggregate on read), caching, or dedicated replicas for hot keys.

**Hash function quality.** Use a good non-cryptographic hash (MurmurHash3, xxHash). A weak hash gives a clumpy ring.

**Coordinating servers.** Every client needs a consistent view of which servers are on the ring. Use a shared config (ZooKeeper, etcd, Consul) or rely on the cache client library's gossip mechanism. Inconsistent views = different clients sending the same key to different servers = cache misses everywhere.

**Replication.** Consistent hashing places one copy. For N replicas, walk the next N servers clockwise. Make sure those servers are in different failure domains (zones, racks) so a correlated failure doesn't take all replicas of a key.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 280" role="img" aria-label="Plain hashing vs consistent hashing when adding a server" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Adding one server: who moves?</text>
  <g transform="translate(0,50)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">hash(k) mod N</text>
    <text x="160" y="18" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">N: 4 → 5</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="35" width="280" height="35" rx="6"/>
      <rect x="20" y="80" width="280" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" font-size="11" text-anchor="middle">
      <text x="160" y="57">~20% of keys stay</text>
      <text x="160" y="102" font-weight="600">~80% of keys move</text>
    </g>
    <text x="160" y="160" text-anchor="middle" fill="var(--sl-color-accent,#3b82f6)" font-size="11">Cache melts. Origin gets flattened.</text>
  </g>
  <g transform="translate(320,50)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Consistent hashing</text>
    <text x="160" y="18" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">N: 4 → 5</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="35" width="280" height="80" rx="6"/>
      <rect x="240" y="80" width="60" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" font-size="11" text-anchor="middle">
      <text x="130" y="60">~80% of keys stay</text>
      <text x="270" y="102" font-weight="600">~1/N moves</text>
    </g>
    <text x="160" y="160" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Cache mostly warm. Origin survives.</text>
  </g>
</svg>

## A short worked example

Suppose you have three cache nodes A, B, C, and the ring positions land them roughly evenly. Keys hash uniformly across the ring; each node ends up serving ~1/3 of them.

You add node D. Without consistent hashing (plain `mod N`), ~75% of keys would move. With consistent hashing and ~150 vnodes per server, D picks up ~1/4 of the keys, drawn proportionally from A, B, and C. The other 3/4 stay where they were. The cache stays mostly warm; the origin sees a manageable spike rather than a meltdown.

That graceful "only 1/N of the keys move on a change" property is the entire reason consistent hashing exists, and it's exactly what an interviewer wants you to articulate.

## What to say in an interview

A one-paragraph version that lands well:

> *"We distribute the cache across N nodes using consistent hashing with ~200 virtual nodes per server. Adding or removing a cache node only invalidates ~1/N of the keys instead of nearly all of them, so we don't melt the origin during scale events. For replication we walk three positions clockwise on the ring, placing each copy on a node in a different availability zone."*

Three concepts (ring, vnodes, replication placement) in one paragraph. That is usually all that's wanted.
