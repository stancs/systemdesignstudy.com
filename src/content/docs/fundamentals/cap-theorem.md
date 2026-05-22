---
title: CAP Theorem & Trade-offs
description: A practical reading of CAP, PACELC, and the consistency vs latency vs availability trade-offs that drive every distributed-system decision.
---

The CAP theorem is the most misquoted idea in distributed systems. Used carelessly, it sounds like a horoscope ("pick two of three"). Used carefully, it gives you a precise vocabulary for the trade-offs you'll defend in nearly every system design interview.

<img src="/diagrams/fundamentals/cap-theorem-1.svg" alt="The CAP triangle and what each pair means under partition" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

## The theorem, stated precisely

Eric Brewer's CAP theorem says that a distributed data store can, in the presence of a **network partition**, provide **at most one** of:

- **C — Consistency:** every read sees the latest committed write.
- **A — Availability:** every request receives a non-error response (though possibly stale).

The third letter, **P — Partition tolerance**, is not really optional in any real distributed system — networks fail, packets drop. So the meaningful choice is between **CP** (consistent under partition, but reject some requests) and **AP** (available under partition, but allow stale or conflicting reads).

A single-node database is "CA" in a trivial sense — there's no partition because there's only one node. The interesting CAP discussion only starts once data lives on more than one machine.

## CP vs AP, in plain terms

**CP systems** would rather return an error than return wrong data. When a partition splits the cluster, the minority side stops accepting writes (and often reads too).

- Examples: Spanner, CockroachDB, etcd, ZooKeeper, HBase.
- Use when: money, inventory, configuration, leader election, anything where wrong > slow > down is *not* acceptable.

**AP systems** would rather keep serving requests than refuse them, accepting that some replicas may diverge and need reconciliation later.

- Examples: Cassandra, DynamoDB (in default mode), Riak, most CDNs and DNS resolvers.
- Use when: social feeds, analytics, view counts, shopping carts, anything where stale > down.

A quick test: if your business answer to *"the system is partitioned — should we serve stale reads or refuse?"* is "serve stale," you want AP. If it is "refuse," you want CP.

## PACELC: the more honest version

CAP only talks about partitions. PACELC (Daniel Abadi) extends it: **if there is a Partition, choose A or C; else, choose L (latency) or C (consistency).**

That second half is the one that matters most days. Even in healthy operation, strong consistency over multiple replicas costs you latency — you have to wait for a quorum, often across availability zones or regions. So a more useful framing is:

- **PA/EL** — Cassandra, Dynamo, most caches: prefer availability under partition, prefer latency in normal operation.
- **PC/EC** — Spanner, CockroachDB, HBase: prefer consistency under partition, prefer consistency in normal operation.

Most production systems are not pure picks. You compose them: a strongly-consistent metadata store (PC/EC) in front of an eventually-consistent bulk store (PA/EL).

<img src="/diagrams/fundamentals/cap-theorem-2.svg" alt="PACELC decision tree for partition vs no-partition trade-offs" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

## Consistency models worth naming

"Strong" and "eventual" are too coarse for a senior interview. Reach for these when relevant:

- **Linearizable / strong consistency** — all clients see operations in a single global order; reads return the latest write. Expensive but simple to reason about.
- **Sequential consistency** — operations appear in *some* total order consistent with each client's program order. Weaker than linearizable; the global "real time" is not respected.
- **Causal consistency** — operations causally related are seen in the same order by everyone; concurrent operations may be seen in different orders. A good middle ground for chats and collaborative editing.
- **Read-your-writes** — a single client always sees its own writes. Crucial for UX (no "did my post save?" loop).
- **Monotonic reads** — once a client has seen a value, it never sees an older one.
- **Eventual consistency** — replicas converge if writes stop; no other guarantees.

Naming the specific guarantee earns more credit than saying "we'll use strong consistency" everywhere.

## Latency vs throughput vs durability

CAP isn't the only set of trade-offs. The other big ones:

**Latency vs throughput.** Batching, async work, and queues raise throughput at the cost of latency. For most user-facing reads you want low latency; for analytics or fan-out you want throughput.

**Latency vs durability.** Synchronous replication to N replicas costs you the slowest replica's latency on every write. Async replication is faster but loses some writes if the leader dies. Quorum (N/2 + 1) is the standard middle ground.

**Cost vs everything.** Stronger guarantees and tighter latencies almost always cost more — more replicas, more nodes, more cross-region traffic. Be willing to say *"this configuration meets our SLO at roughly 2x the cost of the cheaper option."*

## How to use this in an interview

You don't need to recite CAP. You need to make the trade-off explicit at the point of decision:

> *"I'll use a CP store for the payments ledger — we'd rather refuse a transaction than double-charge. The product catalog can be AP because stale price data for a few seconds is acceptable, and we want it to stay available during a regional partition."*

Two sentences, two systems, two correctly-applied trade-offs. That is the bar.

## Common pitfalls

- **Treating CAP as "pick two of three."** P isn't really optional. The choice is C vs A *given* a partition.
- **Calling everything "eventually consistent" without specifying the convergence window.** "Eventually" without a number is a hand-wave.
- **Forgetting that consistency costs latency even without partitions.** PACELC is the half most candidates miss.
- **Designing one consistency model for the whole system.** Real systems mix strong and eventual paths intentionally.

If you can say which letter your store picks *and why the product allows it*, you sound like an engineer who has lived inside these trade-offs — which is the entire point.
