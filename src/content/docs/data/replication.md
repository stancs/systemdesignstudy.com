---
title: Database Replication
description: Single-leader, multi-leader, and leaderless replication — what each model gives you, when to pick which, and how to defend the consistency story.
---

Replication is the practice of keeping multiple copies of your data on different machines. Done well, it gives you durability, availability, and read scalability. Done badly, it gives you data loss, split-brain, and the worst kind of bugs — the silent ones.

There are three replication architectures worth knowing, and every distributed database in the world is some variation of them.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 280" role="img" aria-label="Single-leader replication: one writer, many readers" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Single-leader replication</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="240" y="50" width="160" height="50" rx="10" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="60" y="170" width="140" height="50" rx="10"/>
    <rect x="250" y="170" width="140" height="50" rx="10"/>
    <rect x="440" y="170" width="140" height="50" rx="10"/>
  </g>
  <g fill="currentColor" text-anchor="middle">
    <text x="320" y="72" font-weight="700">Leader</text>
    <text x="320" y="90" font-size="11">accepts writes</text>
    <text x="130" y="192" font-weight="600">Follower 1</text>
    <text x="130" y="210" font-size="11">read replica</text>
    <text x="320" y="192" font-weight="600">Follower 2</text>
    <text x="320" y="210" font-size="11">read replica</text>
    <text x="510" y="192" font-weight="600">Follower 3</text>
    <text x="510" y="210" font-size="11">read replica</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none">
    <path d="M280 100 L150 170"/>
    <path d="M320 100 V170"/>
    <path d="M360 100 L500 170"/>
  </g>
  <g fill="currentColor">
    <polygon points="151,166 154,172 146,170"/>
    <polygon points="316,168 320,174 324,168"/>
    <polygon points="498,168 502,174 506,170"/>
  </g>
  <text x="320" y="252" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.7">Writes go to one node; reads scale by adding followers (with replication lag).</text>
</svg>

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

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 240" role="img" aria-label="Synchronous vs asynchronous replication trade-off" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Sync vs async replication</text>
  <g transform="translate(0,40)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Asynchronous</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="40" y="20" width="80" height="40" rx="8"/>
      <rect x="200" y="20" width="80" height="40" rx="8"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="12">
      <text x="80" y="45">Leader</text>
      <text x="240" y="45">Follower</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M120 40 H200" stroke-dasharray="3 3"/>
    </g>
    <text x="160" y="100" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Leader acks immediately, ships later.</text>
    <text x="160" y="118" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Fast writes, possible data loss on failover.</text>
  </g>
  <g transform="translate(320,40)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Synchronous</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="40" y="20" width="80" height="40" rx="8"/>
      <rect x="200" y="20" width="80" height="40" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="12">
      <text x="80" y="45">Leader</text>
      <text x="240" y="45">Follower</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M120 30 H200"/>
      <path d="M200 50 H120"/>
    </g>
    <g fill="currentColor">
      <polygon points="196,28 202,30 196,32"/>
      <polygon points="124,48 118,50 124,52"/>
    </g>
    <text x="160" y="100" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Leader waits for follower's ack.</text>
    <text x="160" y="118" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">No loss, but slow-follower stalls writes.</text>
  </g>
  <text x="320" y="220" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.7">Most "we can't lose writes" systems use semi-sync: one sync, rest async.</text>
</svg>

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

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 260" role="img" aria-label="Leaderless quorum: N=3, W=2, R=2 writes and reads" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Leaderless quorum (N=3, W=2, R=2)</text>
  <g transform="translate(0,50)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Write (W=2 acks needed)</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="20" width="70" height="35" rx="6"/>
      <rect x="125" y="20" width="70" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="125" y="75" width="70" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="125" y="130" width="70" height="35" rx="6"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="12">
      <text x="55" y="42">Client</text>
      <text x="160" y="42">Node A ✓</text>
      <text x="160" y="97">Node B ✓</text>
      <text x="160" y="152">Node C ✗</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M90 38 H125"/>
      <path d="M90 38 H110 V92 H125"/>
      <path d="M90 38 H110 V147 H125"/>
    </g>
  </g>
  <g transform="translate(320,50)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Read (R=2 consulted)</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="20" width="70" height="35" rx="6"/>
      <rect x="125" y="20" width="70" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="125" y="75" width="70" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="125" y="130" width="70" height="35" rx="6"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="12">
      <text x="55" y="42">Client</text>
      <text x="160" y="42">Node A</text>
      <text x="160" y="97">Node B</text>
      <text x="160" y="152">Node C</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M90 38 H125"/>
      <path d="M90 38 H110 V92 H125"/>
    </g>
  </g>
  <text x="320" y="240" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.7">W + R > N → at least one read node saw the latest write.</text>
</svg>

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
