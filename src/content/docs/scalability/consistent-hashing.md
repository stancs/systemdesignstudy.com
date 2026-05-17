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

## A short worked example

Suppose you have three cache nodes A, B, C, and the ring positions land them roughly evenly. Keys hash uniformly across the ring; each node ends up serving ~1/3 of them.

You add node D. Without consistent hashing (plain `mod N`), ~75% of keys would move. With consistent hashing and ~150 vnodes per server, D picks up ~1/4 of the keys, drawn proportionally from A, B, and C. The other 3/4 stay where they were. The cache stays mostly warm; the origin sees a manageable spike rather than a meltdown.

That graceful "only 1/N of the keys move on a change" property is the entire reason consistent hashing exists, and it's exactly what an interviewer wants you to articulate.

## What to say in an interview

A one-paragraph version that lands well:

> *"We distribute the cache across N nodes using consistent hashing with ~200 virtual nodes per server. Adding or removing a cache node only invalidates ~1/N of the keys instead of nearly all of them, so we don't melt the origin during scale events. For replication we walk three positions clockwise on the ring, placing each copy on a node in a different availability zone."*

Three concepts (ring, vnodes, replication placement) in one paragraph. That is usually all that's wanted.
