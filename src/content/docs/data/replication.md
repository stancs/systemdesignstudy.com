---
title: Database Replication
description: Single-leader, multi-leader, and leaderless replication — what each model gives you, when to pick which, and how to defend the consistency story.
---

Replication is the practice of keeping multiple copies of your data on different machines. Done well, it gives you durability, availability, and read scalability. Done badly, it gives you data loss, split-brain, and the worst kind of bugs — the silent ones.

There are three replication architectures worth knowing, and every distributed database in the world is some variation of them.

## Single-leader (primary-replica)

One node is the **leader** (also called primary, master, or writer). All writes go to it. The leader streams its changes to one or more **followers** (replicas, secondaries), which apply them in order.

This is what most teams reach for first because it is the simplest model that gets you anywhere useful.

**What you get**

- **Write simplicity.** One node owns the truth at any moment. No conflict resolution needed.
- **Read scaling.** Followers serve reads. If 95% of your traffic is reads, you've effectively scaled by adding follower nodes.
- **Mature tooling.** Postgres streaming replication, MySQL binlog replication, MongoDB replica sets, Redis primary/replica.

**What it costs**

- **Single writer is your bottleneck.** Once the leader saturates, your only options are vertical scaling, [sharding](/data/sharding/), or moving to a different model.
- **Replication lag.** Followers are always slightly behind. The lag is usually milliseconds but can spike under load.
- **Failover is non-trivial.** If the leader dies, you have to promote a follower. Choosing *which* follower (the most up-to-date one) and not double-promoting is the entire reason tools like Patroni, Orchestrator, and managed RDS exist.

### Synchronous vs asynchronous

A critical sub-decision within single-leader.

**Asynchronous.** Leader writes locally, returns success, then ships the change to followers in the background. Low write latency, but if the leader dies before replicating a recent write, that write is lost.

**Synchronous (or semi-synchronous).** Leader waits for at least one follower to confirm before returning. No data loss on failover, but write latency is the slowest replica's, and a slow follower can block all writes.

The common compromise: **semi-synchronous** — require one synchronous follower (zero data loss) and let the rest replicate asynchronously (good performance). This is the right default for most "we can't lose writes" workloads.

### Read-your-writes consistency

Replication lag breaks an obvious UX assumption: a user posts something, immediately reloads, and the post isn't there yet because the read went to a stale follower. Three fixes:

- **Route reads after writes to the leader** for a short window.
- **Sticky sessions** — route all of a user's reads to the same replica or to the leader.
- **Read tokens** — the write returns a logical sequence number; subsequent reads include it and wait until the follower has caught up.

Mention this if the prompt involves any kind of user-generated content.

## Multi-leader

Multiple nodes accept writes; they replicate to each other. The classic use case is a **multi-region active-active** deployment: every region has a local leader, and writes from anywhere are eventually visible everywhere.

**What you get**

- **Low write latency in every region.** No cross-region round trip.
- **Local survival.** A regional outage doesn't stop local writes.

**What it costs**

- **Conflicts.** Two leaders can accept conflicting writes to the same row. You need a conflict resolution strategy: last-write-wins (lossy), CRDTs (designed to merge correctly), application-level reconciliation (you write the merge logic).
- **Operational complexity.** Topology, conflict tracking, monitoring. Multi-leader is the bug factory of replication models.

Most teams avoid full multi-leader and approximate it by **sharding by region** instead — each shard has a single leader in its home region, and cross-region requests pay the round trip. This is much simpler and rarely worse in practice.

When to pick multi-leader anyway: you must accept local writes during a network partition, the data is collaborative-edit-style (Google Docs, Notion), or your store is something explicitly designed for it (Cassandra/Dynamo are technically leaderless, which we'll get to next).

## Leaderless (quorum-based)

No leader. Clients send each write to multiple nodes; reads also query multiple nodes; the system uses a **quorum** to decide which value wins. This is what Cassandra, DynamoDB, and Riak do, descended from the Dynamo paper.

The standard parameters:

- **N** — total number of replicas.
- **W** — writes acknowledged before success.
- **R** — reads consulted before returning.

If **W + R > N**, you are guaranteed to read at least one node that saw the latest write — that's quorum consistency. Common settings: N=3, W=2, R=2 (strong-ish) or N=3, W=1, R=1 (fast, eventually consistent).

**What you get**

- **No single point of failure.** Any node can serve any read or write.
- **Tunable consistency.** Choose the W and R that suit each operation.
- **Smooth degradation.** Losing one replica is invisible at the quorum level.

**What it costs**

- **Conflicts again** — two writes to the same key from different clients can both succeed without coordination. Conflict resolution (last-write-wins, vector clocks, CRDTs) is required.
- **Read amplification.** Every read fans out to R nodes.
- **Anti-entropy overhead.** Background processes (read repair, hinted handoff, Merkle-tree-based repairs) keep replicas converged.

Pick leaderless when you have write-heavy, key-shaped workloads that need to survive partial failures with no operator intervention.

## Replication factor and topology

Independent of the architecture, you choose:

- **Replication factor (RF).** How many copies. RF=3 is the typical sweet spot — survives the loss of any single node and most double-failure scenarios.
- **Placement.** Replicas across availability zones (survives a zone failure), or across regions (survives a region failure). Cross-region replication adds latency proportional to physical distance.

Quorum math becomes more interesting across regions: a quorum of 2 out of 3 with replicas in three different regions means every write pays a cross-region round trip. Real systems often run **5 replicas across 3 zones** to balance durability and latency.

## How to talk about replication in an interview

Two or three sentences usually suffices:

> *"The primary database is Postgres with one leader and two followers, all in the same region. Followers handle read traffic. We use semi-synchronous replication to the nearest follower so no committed write is lost on a leader failure. For multi-region we plan to shard by user geography rather than running multi-leader — much simpler conflict story."*

If the prompt is at very high scale or has hard availability targets, the answer moves toward distributed SQL (Spanner, CockroachDB) or leaderless NoSQL (Cassandra, DynamoDB), and you justify the move with numbers.

## Common pitfalls

**Treating async replication as durable.** Followers are not backups. A bug or `DROP TABLE` replicates to them in milliseconds. Keep proper point-in-time backups separate.

**Promoting the wrong follower.** Always promote the one furthest along in the replication log. Tools that automate this exist for a reason.

**Forgetting that reads go to lagging replicas.** If you sprinkle reads across followers, you have *implicitly* picked eventual consistency for those reads. Make that explicit.

**Going multi-leader by accident.** Cross-region active-active sounds great until you've never resolved a write conflict. Pick it deliberately, not by default.

Replication is one of the topics where naming the *specific* model and the *specific* trade-off is dramatically more impressive than a generic "we'll replicate the database." Be specific.
