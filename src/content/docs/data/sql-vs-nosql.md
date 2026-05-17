---
title: SQL vs NoSQL
description: A pragmatic comparison of relational and non-relational databases — what each model actually gives you, when to pick which, and what to say in the interview.
---

The "SQL vs NoSQL" question is unavoidable in system design interviews, and most candidates answer it badly — usually by mentioning "scale" without specifying what scale means or why one side of the divide handles it better. The honest version is: relational and non-relational databases solve different problems, and modern systems often use both.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 240" role="img" aria-label="A relational model with rows, columns, and a join" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:12px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">SQL: structured rows joined by keys</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="20" y="50" width="220" height="170" rx="8"/>
    <rect x="400" y="50" width="220" height="170" rx="8"/>
  </g>
  <g fill="currentColor">
    <text x="130" y="74" text-anchor="middle" font-weight="700">users</text>
    <text x="510" y="74" text-anchor="middle" font-weight="700">orders</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none">
    <line x1="30" y1="90" x2="230" y2="90"/>
    <line x1="100" y1="90" x2="100" y2="215"/>
    <line x1="170" y1="90" x2="170" y2="215"/>
    <line x1="410" y1="90" x2="610" y2="90"/>
    <line x1="470" y1="90" x2="470" y2="215"/>
    <line x1="540" y1="90" x2="540" y2="215"/>
  </g>
  <g fill="currentColor" font-size="11">
    <text x="35" y="106">id</text>
    <text x="105" y="106">email</text>
    <text x="175" y="106">name</text>
    <text x="35" y="130">1</text>
    <text x="105" y="130">a@x</text>
    <text x="175" y="130">Ann</text>
    <text x="35" y="154">2</text>
    <text x="105" y="154">b@x</text>
    <text x="175" y="154">Bob</text>
    <text x="35" y="178">3</text>
    <text x="105" y="178">c@x</text>
    <text x="175" y="178">Cleo</text>
    <text x="415" y="106">id</text>
    <text x="475" y="106">user_id</text>
    <text x="545" y="106">total</text>
    <text x="415" y="130">11</text>
    <text x="475" y="130">2</text>
    <text x="545" y="130">$19</text>
    <text x="415" y="154">12</text>
    <text x="475" y="154">1</text>
    <text x="545" y="154">$48</text>
    <text x="415" y="178">13</text>
    <text x="475" y="178">2</text>
    <text x="545" y="178">$5</text>
  </g>
  <path d="M240 130 H400" fill="none" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2" stroke-dasharray="5 4"/>
  <text x="320" y="124" text-anchor="middle" fill="var(--sl-color-accent,#3b82f6)" font-weight="600" font-size="11">JOIN on user_id</text>
</svg>

## What "SQL" really means

A SQL (relational) database stores data in tables of rows and columns, with a fixed schema, strong ACID guarantees, and a rich query language. Examples: **PostgreSQL, MySQL, Oracle, SQL Server**, plus distributed variants like **CockroachDB, Spanner, YugabyteDB**.

What you actually get:

- **ACID transactions** — Atomicity, Consistency, Isolation, Durability across multiple rows and tables.
- **Joins** — combine data from multiple tables in a single query.
- **Strict schemas** — every row in a table has the same columns and types.
- **Mature tooling** — query optimizers, replication, backups, observability.

What it costs:

- **Vertical scaling first.** Traditional single-leader Postgres maxes out around 50–100k QPS on a beefy box; you scale by sharding (manually or with extensions like Citus) or moving to a distributed SQL engine.
- **Schema changes are expensive** at large table sizes. Adding a column to a billion-row table without an outage takes planning.
- **Write throughput** is the usual bottleneck. Replicas help reads, not writes.

## What "NoSQL" really means

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 280" role="img" aria-label="The four NoSQL data models with example shapes" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:12px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">The four NoSQL data models</text>
  <g transform="translate(20,45)">
    <rect width="140" height="210" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="70" y="24" text-anchor="middle" fill="currentColor" font-weight="700">Key-value</text>
    <text x="70" y="42" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">Redis · DynamoDB</text>
    <g fill="currentColor" font-size="11">
      <text x="14" y="74">"user:42" → {…}</text>
      <text x="14" y="96">"sess:x9" → {…}</text>
      <text x="14" y="118">"cart:7" → […]</text>
    </g>
    <text x="70" y="170" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">O(1) lookups</text>
    <text x="70" y="186" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">no joins</text>
  </g>
  <g transform="translate(170,45)">
    <rect width="140" height="210" rx="10" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2"/>
    <text x="70" y="24" text-anchor="middle" fill="currentColor" font-weight="700">Document</text>
    <text x="70" y="42" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">MongoDB · Firestore</text>
    <g fill="currentColor" font-size="11">
      <text x="14" y="70">{ "id":1,</text>
      <text x="14" y="86">  "name":"Ann",</text>
      <text x="14" y="102">  "tags":[…],</text>
      <text x="14" y="118">  "addr":{…} }</text>
    </g>
    <text x="70" y="170" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">flexible schema</text>
    <text x="70" y="186" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">field queries</text>
  </g>
  <g transform="translate(320,45)">
    <rect width="140" height="210" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="70" y="24" text-anchor="middle" fill="currentColor" font-weight="700">Wide-column</text>
    <text x="70" y="42" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">Cassandra · Scylla</text>
    <g fill="currentColor" font-size="10">
      <text x="14" y="70">user_id | ts | val</text>
      <text x="14" y="86">42 | t1 | hi</text>
      <text x="14" y="102">42 | t2 | bye</text>
      <text x="14" y="118">42 | t3 | ok</text>
    </g>
    <text x="70" y="170" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">huge write rates</text>
    <text x="70" y="186" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">key partition</text>
  </g>
  <g transform="translate(470,45)">
    <rect width="150" height="210" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="75" y="24" text-anchor="middle" fill="currentColor" font-weight="700">Graph</text>
    <text x="75" y="42" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">Neo4j · Neptune</text>
    <g transform="translate(20,70)">
      <circle cx="20" cy="20" r="10" fill="none" stroke="currentColor" stroke-width="1.5"/>
      <circle cx="80" cy="20" r="10" fill="none" stroke="currentColor" stroke-width="1.5"/>
      <circle cx="20" cy="60" r="10" fill="none" stroke="currentColor" stroke-width="1.5"/>
      <circle cx="80" cy="60" r="10" fill="none" stroke="currentColor" stroke-width="1.5"/>
      <line x1="30" y1="20" x2="70" y2="20" stroke="currentColor"/>
      <line x1="20" y1="30" x2="20" y2="50" stroke="currentColor"/>
      <line x1="80" y1="30" x2="80" y2="50" stroke="currentColor"/>
      <line x1="30" y1="60" x2="70" y2="60" stroke="currentColor"/>
      <line x1="28" y1="28" x2="72" y2="52" stroke="currentColor"/>
    </g>
    <text x="75" y="170" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">traversal queries</text>
    <text x="75" y="186" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">friends-of-friends</text>
  </g>
</svg>

"NoSQL" is a four-way bucket of fundamentally different data models:

- **Key-value** (Redis, Memcached, DynamoDB, etcd). Just a giant distributed hash map. The simplest model and often the fastest.
- **Document** (MongoDB, Couchbase, Firestore). JSON-shaped documents, flexible schema, queries by field.
- **Wide-column** (Cassandra, ScyllaDB, HBase, Bigtable). Tables with rows and dynamic columns; queries are designed around a known partition key.
- **Graph** (Neo4j, JanusGraph, Amazon Neptune). Nodes and edges, queried via traversal languages like Cypher or Gremlin.

What you typically get across the family:

- **Horizontal scaling as a primary feature.** Partitioning is built in; adding nodes increases capacity linearly within limits.
- **Flexible schemas.** You can evolve the shape of stored data without a migration step.
- **Predictable, low-latency single-key access** — orders of magnitude faster than complex SQL for the same simple lookup.

What it costs:

- **Weaker transactional guarantees.** Most NoSQL stores offer single-key atomicity but not multi-key or cross-table transactions (some now offer them, but at a latency cost).
- **No joins.** You denormalize at write time or join in the application.
- **Query patterns must be known up front.** Cassandra in particular is designed around access patterns, not the data shape.
- **Operational complexity.** Tuning, repairs, compactions, and partition design are all your job.

## The actual decision

Forget "scale." Make the decision on the **access pattern** and the **consistency requirements**.

Reach for **SQL** when:

- You have many entity types with relationships you'll want to query across (orders, users, products, payments).
- You need transactions spanning multiple rows or tables.
- Access patterns will evolve and you don't want to predetermine queries.
- Strong consistency is non-negotiable somewhere in the system.

Reach for **NoSQL** when:

- Access is overwhelmingly **single-key or single-partition** lookups.
- Write throughput exceeds what a single SQL leader can handle (~50–100k writes/sec).
- The data model is genuinely document- or graph-shaped, and forcing it into rows would be painful.
- You'd rather denormalize at write time than join at read time.

In real systems you frequently use both: a SQL store as the system of record + a NoSQL store as a read-optimized cache or denormalized view.

## Concrete patterns by domain

**E-commerce checkout** — SQL. Money, inventory, ACID. Don't be clever.

**Product catalog reads at 100k QPS** — Postgres as source of truth + Redis or DynamoDB as a cached read layer.

**Activity feeds and timelines** — wide-column (Cassandra, ScyllaDB). Append-only writes, key by user, time-ordered. A canonical fit.

**User session store** — key-value (Redis). Tiny payloads, microsecond reads, TTL-based expiry.

**Social graph** — graph database for traversal queries (friends-of-friends), often layered with denormalized lookup tables for hot paths.

**Real-time analytics / logs** — columnar (ClickHouse, BigQuery, Druid), which is a third category outside the SQL/NoSQL binary but worth knowing exists.

## "Modern" complications

A few things worth knowing because interviewers do bring them up:

- **NewSQL / distributed SQL.** CockroachDB, Spanner, YugabyteDB, Vitess. SQL semantics with horizontal scale via Paxos/Raft and sharded storage. Higher write latency than single-leader SQL; better than NoSQL for transactional workloads at scale.
- **NoSQL with transactions.** DynamoDB Transactions, MongoDB multi-document transactions, FoundationDB. Real but with cost — usually higher latency and limited scope.
- **Multi-model databases.** Cosmos DB, ArangoDB, Couchbase. One engine, multiple data models. Convenient operationally; rarely the best fit for any single workload.

## Common interview pitfalls

**Picking NoSQL "for scale" on a moderate workload.** Postgres on a single beefy machine plus read replicas handles enormous traffic. Justify NoSQL with an actual number.

**Picking SQL because it's familiar.** If the access pattern is "give me everything for user 42," a key-value store is dramatically simpler and faster.

**Forgetting the consistency story.** Whichever you pick, name the consistency level. See [CAP Theorem](/fundamentals/cap-theorem/).

**Treating the choice as monolithic.** Real systems use multiple stores, each for its strength.

## What to say in an interview

A confident one-liner that earns credit:

> *"The source of truth is Postgres — we have transactions across users, orders, and payments. Read-heavy lookups (product detail, session, feature flags) are served from Redis with cache-aside. The activity feed lives in Cassandra because writes go straight to a user-keyed timeline and we never need cross-user joins on it."*

Three stores, three reasons, each tied to an access pattern. That is the bar.
