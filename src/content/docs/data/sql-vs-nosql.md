---
title: SQL vs NoSQL
description: A pragmatic comparison of relational and non-relational databases — what each model actually gives you, when to pick which, and what to say in the interview.
---

The "SQL vs NoSQL" question is unavoidable in system design interviews, and most candidates answer it badly — usually by mentioning "scale" without specifying what scale means or why one side of the divide handles it better. The honest version is: relational and non-relational databases solve different problems, and modern systems often use both.

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
