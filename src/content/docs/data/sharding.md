---
title: Sharding & Partitioning
description: Horizontal partitioning, choosing a shard key, the difference between range, hash, and directory sharding — and the operational pain you should be prepared to discuss.
---

Once a single database leader cannot keep up with the write rate (or once the dataset no longer fits on a single host), you have to split it. **Sharding** (also called horizontal partitioning) is the practice of splitting one logical table across many physical nodes, each holding a subset of the rows. Every query then has to figure out *which shard* to talk to.

Sharding is one of the highest-leverage, highest-cost moves in system design. You bring it up only when you've justified it with numbers.

## Vertical vs horizontal partitioning

Worth distinguishing once:

- **Vertical partitioning** — different *columns* live on different machines (or in different tables). E.g., move `user_profile_bio` to a separate table because it's rarely accessed.
- **Horizontal partitioning (sharding)** — different *rows* live on different machines. Most of the time when people say "partitioning" or "sharding," they mean this.

When in doubt, "sharding" = horizontal, and that's what the rest of this page is about.

## The shard key is the entire decision

Picking the **shard key** is the single most consequential decision in a sharded system. Every read and write either:

- Hits a single shard (cheap, scales linearly), because it knows the shard key.
- Has to fan out across all shards (expensive, doesn't scale), because it doesn't.

The right shard key is the one your most important queries already include. Picking based on a less-frequent dimension is the classic mistake that turns "we sharded for scale" into "every query is now a scatter-gather."

A useful test: write down your top five queries. Look at the `WHERE` clauses. The column that appears most often is your shard key candidate.

## Sharding strategies

Three classic strategies, plus one modern hybrid.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 260" role="img" aria-label="Range vs hash vs directory sharding placement" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:12px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Three ways to assign rows to shards</text>
  <g transform="translate(0,45)">
    <text x="105" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Range</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="20" width="170" height="35" rx="6"/>
      <rect x="20" y="65" width="170" height="35" rx="6"/>
      <rect x="20" y="110" width="170" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="105" y="42">user_id 1 – 1M  → Shard A</text>
      <text x="105" y="87">user_id 1M – 2M  → Shard B</text>
      <text x="105" y="132">user_id 2M+  → Shard C (hot)</text>
    </g>
    <text x="105" y="180" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Easy range scans;</text>
    <text x="105" y="196" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">hot tail with sequential IDs.</text>
  </g>
  <g transform="translate(220,45)">
    <text x="105" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Hash</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="20" width="170" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="20" y="65" width="170" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="20" y="110" width="170" height="35" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="105" y="42">hash(id) mod 3 = 0  → A</text>
      <text x="105" y="87">hash(id) mod 3 = 1  → B</text>
      <text x="105" y="132">hash(id) mod 3 = 2  → C</text>
    </g>
    <text x="105" y="180" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Even distribution;</text>
    <text x="105" y="196" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">range queries scatter.</text>
  </g>
  <g transform="translate(440,45)">
    <text x="100" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Directory</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="10" y="20" width="180" height="125" rx="6"/>
    </g>
    <g fill="currentColor" font-size="11">
      <text x="25" y="42">tenant_42 → Shard A</text>
      <text x="25" y="66">tenant_7  → Shard B</text>
      <text x="25" y="90">tenant_99 → Shard A</text>
      <text x="25" y="114">tenant_BIG → dedicated</text>
      <text x="25" y="138" opacity="0.7" font-style="italic">lookup service</text>
    </g>
    <text x="100" y="180" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Maximum flexibility;</text>
    <text x="100" y="196" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">directory is a hot dependency.</text>
  </g>
</svg>

### Range sharding

Rows are partitioned by ranges of the key. `user_id 1–1M` lives on shard A, `1M–2M` on shard B, and so on.

- **Pros:** Range scans within a partition are efficient. Easy to reason about.
- **Cons:** **Hot spots.** If new users are heavier than old users, the latest range gets all the writes. Monotonically increasing keys (timestamps, auto-increment IDs) are a disaster as range shard keys.

Used by: HBase, Bigtable, CockroachDB (which rebalances ranges dynamically).

### Hash sharding

Apply a hash function to the shard key and partition by the hash. `hash(user_id) mod N` picks the shard.

- **Pros:** Even distribution by default. No hot spots from key sequencing.
- **Cons:** Range queries on the shard key become scatter-gather. Resharding (changing N) is painful because every row's destination changes.

This is where [Consistent Hashing](/scalability/consistent-hashing/) earns its keep — by minimizing the rows that move when N changes.

Used by: Cassandra, DynamoDB, most key-value stores.

### Directory-based sharding

A lookup table maps each key (or key range) to its shard. The mapping is itself stored somewhere — usually a small, replicated metadata service.

- **Pros:** Maximum flexibility. You can place specific tenants on specific shards, isolate a hot customer, or migrate keys without resharding everything.
- **Cons:** The directory becomes a critical-path service. You pay an extra hop on each lookup unless clients cache the mapping.

Used by: Vitess, many bespoke multi-tenant systems.

### Geographic / tenant-based sharding

A variant of directory sharding where the shard key is a real-world dimension — country, customer, organization. Keeps related data colocated for both performance and compliance.

Used by: most B2B SaaS at scale (one shard per large customer), apps with strict data-residency requirements.

## What changes when you shard

Sharding is not free. A non-exhaustive list of things you now have to handle:

**Joins across shards.** They become expensive scatter-gather. Mitigate by **denormalizing** so joins don't cross shards, or by colocating related data on the same shard (e.g., shard `orders` by the same `user_id` that shards `users`).

**Transactions across shards.** The cheap single-shard transaction is fine. Cross-shard transactions require distributed commit (two-phase commit or modern equivalents) and are an order of magnitude slower. Design schemas so most transactions stay on one shard.

**Unique constraints.** A `UNIQUE(email)` across shards is hard. Either pick a shard key that makes the uniqueness scope a single shard, or maintain a separate "uniqueness service" — usually a small, separately-keyed lookup table.

**Rebalancing.** Eventually a shard fills up, gets too hot, or a customer outgrows it. You need a story: consistent hashing with virtual nodes, range splits (Bigtable/Cockroach style), or a directory with online moves (Vitess style). Whatever you pick, mention it.

**Operational overhead.** Backups, schema migrations, monitoring — all of it gets N times as much work. Tooling that hides this (Vitess, Citus, managed Spanner) is worth its cost.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 240" role="img" aria-label="Cross-shard query before and after colocating related tables" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:12px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Colocate by the same shard key to avoid scatter-gather</text>
  <g transform="translate(0,45)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Without colocation</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="30" y="20" width="80" height="50" rx="6"/>
      <rect x="120" y="20" width="80" height="50" rx="6"/>
      <rect x="210" y="20" width="80" height="50" rx="6"/>
      <rect x="30" y="90" width="80" height="50" rx="6"/>
      <rect x="120" y="90" width="80" height="50" rx="6"/>
      <rect x="210" y="90" width="80" height="50" rx="6"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="70" y="40" font-weight="600">users A</text>
      <text x="160" y="40" font-weight="600">users B</text>
      <text x="250" y="40" font-weight="600">users C</text>
      <text x="70" y="110" font-weight="600">orders A</text>
      <text x="160" y="110" font-weight="600">orders B</text>
      <text x="250" y="110" font-weight="600">orders C</text>
    </g>
    <g stroke="var(--sl-color-accent,#3b82f6)" stroke-width="1.5" stroke-dasharray="3 3" fill="none">
      <path d="M70 70 V90"/>
      <path d="M70 70 V80 H160 V90"/>
      <path d="M70 70 V80 H250 V90"/>
    </g>
    <text x="160" y="170" text-anchor="middle" fill="var(--sl-color-accent,#3b82f6)" font-size="11">JOIN touches all shards</text>
  </g>
  <g transform="translate(320,45)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">With colocation by user_id</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="30" y="20" width="80" height="50" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="120" y="20" width="80" height="50" rx="6"/>
      <rect x="210" y="20" width="80" height="50" rx="6"/>
      <rect x="30" y="90" width="80" height="50" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="120" y="90" width="80" height="50" rx="6"/>
      <rect x="210" y="90" width="80" height="50" rx="6"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="70" y="40" font-weight="600">users A</text>
      <text x="160" y="40" font-weight="600">users B</text>
      <text x="250" y="40" font-weight="600">users C</text>
      <text x="70" y="110" font-weight="600">orders A</text>
      <text x="160" y="110" font-weight="600">orders B</text>
      <text x="250" y="110" font-weight="600">orders C</text>
    </g>
    <g stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2" fill="none">
      <path d="M70 70 V90"/>
    </g>
    <text x="160" y="170" text-anchor="middle" fill="var(--sl-color-accent,#3b82f6)" font-size="11">JOIN stays on one shard</text>
  </g>
</svg>

## Hot keys

Even with a well-chosen shard key, individual keys can be hot. The classic case: a celebrity user with 10M followers, on a `user_id`-sharded `followers` table. One shard, one key, hundreds of thousands of reads per second.

Standard mitigations:

- **Caching.** Put a CDN or per-key cache in front of the hot shard.
- **Replication.** Run extra read replicas of the hot shard.
- **Key splitting.** Append a random suffix to writes and aggregate across them on read.
- **Special-case routing.** Route known-hot users to a dedicated tier.

In interviews, mentioning the hot-key problem before the interviewer asks is a strong signal.

## Sharding vs scaling up

Always compare:

- Vertical scaling (bigger box). Cheap-ish operationally, real ceiling.
- Read replicas (followers). Scales reads but not writes.
- Sharding. Scales writes but multiplies operational cost.

Senior candidates don't immediately reach for sharding. They show the math: *"At our estimated 50k writes/sec, a single Postgres leader on a modern instance is fine — we'd plan to revisit when sustained writes exceed 30k/sec, which our growth model puts about 18 months out."*

## What to say in an interview

A clean way to introduce sharding:

> *"For the messages table we shard by `conversation_id` using consistent hashing across 32 shards. The dominant access pattern — 'load the last N messages in this conversation' — stays on one shard. Cross-conversation queries are rare and handled by a denormalized search index in Elasticsearch. We use virtual nodes so that adding shards moves roughly 1/N of the data instead of half."*

Three concepts (shard key, strategy, rebalancing), one for each sentence. That is the bar.
