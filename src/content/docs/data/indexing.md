---
title: Database Indexing
description: How indexes work, why B-trees and LSM-trees dominate, when indexes hurt instead of help, and the indexing decisions worth defending in an interview.
---

An index is a separate data structure the database maintains so that queries don't have to scan an entire table. Without indexes, a `WHERE user_id = 42` on a billion-row table reads a billion rows. With the right index it reads two or three pages from disk. That difference — six or seven orders of magnitude — is why indexes deserve their own mental model rather than being a vague "make queries fast" handwave.

## The two index structures that matter

**B-trees (and B+ trees).** Balanced trees with all data (or pointers to data) in the leaves. They support point lookups, range scans, and ordered iteration in `O(log n)`. Updates rebalance the tree. This is what every traditional SQL database uses by default — Postgres, MySQL/InnoDB, SQL Server, Oracle.

**LSM-trees (log-structured merge-trees).** Writes go into an in-memory buffer, then are flushed to immutable on-disk files, which are periodically merged ("compaction") into larger ones. This is what most modern NoSQL stores use — Cassandra, ScyllaDB, RocksDB, LevelDB, HBase. Writes are very fast (append-only); reads may need to check multiple files and are slower than B-tree reads.

A useful framing:

- **B-trees** optimize for **read-heavy** workloads with mixed reads and writes.
- **LSM-trees** optimize for **write-heavy** workloads where reads can tolerate slightly more latency.

In an interview, you don't have to defend the implementation in detail. You should be able to say *"Postgres uses B-trees, so range scans on sorted columns are cheap; Cassandra uses LSM-trees, so high write throughput is easier but read amplification is real."*

## Primary index vs secondary index

The **primary index** stores the actual row data (or in MySQL/InnoDB and Cassandra, defines the on-disk order of rows). Every table has one — explicitly or implicitly.

A **secondary index** stores a smaller key plus a pointer back to the primary row. Looking up by a secondary index always involves at least two reads: one to find the pointer, one to fetch the row.

Consequence: secondary indexes are not free. Each one adds disk space and slows every write to the table by the cost of updating that index. A table with eight indexes pays for those eight indexes on every insert and update. Indexes should pay rent.

## Index types you should know

- **Hash index.** O(1) lookups, no range support. Use when you only ever query by exact key (e.g., a key-value workload).
- **B-tree index.** Ordered, supports equality and range. The default.
- **Composite (multi-column) index.** Indexes on `(a, b, c)`. Useful for queries that filter or sort on a prefix of the columns. Order matters: `(country, city, zip)` supports queries on `country` alone or `country + city`, but not on `city` alone.
- **Covering index.** A composite index that includes all the columns the query needs. The database can answer the query from the index alone, never touching the table — this is huge for hot read paths.
- **Partial index.** Only indexes rows matching a predicate (e.g., `WHERE status = 'active'`). Smaller, faster, cheaper to maintain.
- **Expression / functional index.** Indexes on an expression like `LOWER(email)`. Use when queries always normalize.
- **Inverted index.** A mapping from each term to the documents containing it. The core data structure inside search engines (Elasticsearch, Lucene). Worth knowing if the prompt involves text search.
- **Geospatial index.** R-tree, geohash, or quadtree-based. Required for "what's near me?" queries.

## Cardinality and selectivity

The single most important property of an index is how many **distinct values** it has relative to table size.

- A column with very high cardinality (`user_id`, `email`, `request_id`) makes an excellent index.
- A column with very low cardinality (`is_active`, `country` with only 4 distinct values) makes a poor standalone index — the query planner often ignores it because reading 25% of the table sequentially is faster than 25% of the rows via random-access seeks.

This is why **the column you filter on first is usually the most selective one**, and why composite indexes are designed leading-most selective.

## Reads vs writes — the eternal trade-off

Adding indexes makes reads faster and writes slower. Every update has to touch every relevant index. The classic mistake is to add an index for every column "just in case." A more disciplined approach:

1. Identify the query patterns the table has to serve. (Three to five real queries is usually enough.)
2. For each, pick the smallest set of indexes that lets the planner answer it efficiently.
3. Drop any index that doesn't have a query using it.
4. Re-evaluate after a few weeks of production data.

Most databases expose statistics on index usage; use them.

## The query planner

Indexes only help if the planner uses them. A few traps that defeat indexes even when they exist:

- **Functions on indexed columns.** `WHERE LOWER(email) = 'a@b.com'` will not use a plain index on `email`. Use a functional index or normalize at write time.
- **Implicit type casts.** `WHERE phone = 12345` against a `phone VARCHAR` column may scan the table.
- **`OR` across columns.** Often produces a sequential scan; rewrite as `UNION ALL`.
- **Leading wildcards.** `LIKE '%foo'` cannot use a B-tree index. `LIKE 'foo%'` can.
- **`NOT IN` and negations.** Frequently turn into scans; reformulate when possible.

Run `EXPLAIN` (or your database's equivalent) when in doubt. In an interview you can name-drop this: *"I'd run `EXPLAIN ANALYZE` to confirm the planner is actually using the composite index."*

## Indexes at scale

When a table gets big enough, the index itself becomes a problem:

- **Memory pressure.** A working B-tree wants its top levels in RAM. If your hot index can't fit, every query pays disk-seek latency.
- **Write amplification.** LSM-tree compaction reads and rewrites data many times; a write-heavy table with many indexes can spend the majority of disk IOPS on compaction.
- **Lock contention.** Hot index pages become a bottleneck in OLTP workloads, especially for monotonically increasing keys (timestamps, auto-increment IDs) — the leftmost leaf becomes a contention point. Mitigation: use ULIDs, snowflake-style IDs, or partitioned indexes.

If you mention these, mention the fix in the same breath. Senior candidates don't list problems without remedies.

## What to say in an interview

When you sketch the data model, name the indexes alongside it:

> *"`posts` is keyed by `(user_id, created_at)` so the dominant query — 'last N posts by user' — is an ordered range scan, and we have a covering index on `(user_id, created_at, post_id)` so the feed read never has to touch the heap. We add a partial index on `(created_at)` only over the last seven days for the trending query, which is cheaper to maintain than a full one."*

That single paragraph tells the interviewer you've thought about the queries, the index structure, the cost of maintenance, and the cost of reads. That is what indexing earns its place in the interview for.
