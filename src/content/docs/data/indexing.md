---
title: Database Indexing
description: How indexes work, why B-trees and LSM-trees dominate, when indexes hurt instead of help, and the indexing decisions worth defending in an interview.
---

An index is a separate data structure the database maintains so that queries don't have to scan an entire table. Without indexes, a `WHERE user_id = 42` on a billion-row table reads a billion rows. With the right index it reads two or three pages from disk. That difference — six or seven orders of magnitude — is why indexes deserve their own mental model rather than being a vague "make queries fast" handwave.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 300" role="img" aria-label="B-tree vs LSM-tree index structures side by side" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:12px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">B-tree vs LSM-tree</text>
  <g transform="translate(20,40)">
    <text x="145" y="0" text-anchor="middle" fill="currentColor" font-weight="600">B-tree (Postgres, MySQL)</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="115" y="20" width="60" height="28" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="30" y="80" width="60" height="28" rx="6"/>
      <rect x="115" y="80" width="60" height="28" rx="6"/>
      <rect x="200" y="80" width="60" height="28" rx="6"/>
      <rect x="0" y="140" width="50" height="28" rx="6"/>
      <rect x="60" y="140" width="50" height="28" rx="6"/>
      <rect x="120" y="140" width="50" height="28" rx="6"/>
      <rect x="180" y="140" width="50" height="28" rx="6"/>
      <rect x="240" y="140" width="50" height="28" rx="6"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="145" y="39">root</text>
      <text x="60" y="98">10..</text>
      <text x="145" y="98">..50..</text>
      <text x="230" y="98">..99</text>
      <text x="25" y="158">1-9</text>
      <text x="85" y="158">10-49</text>
      <text x="145" y="158">50-69</text>
      <text x="205" y="158">70-89</text>
      <text x="265" y="158">90-99</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M145 48 V60 H60 V80"/>
      <path d="M145 48 V60 H145 V80"/>
      <path d="M145 48 V60 H230 V80"/>
      <path d="M60 108 V120 H25 V140"/>
      <path d="M60 108 V120 H85 V140"/>
      <path d="M145 108 V120 H145 V140"/>
      <path d="M230 108 V120 H205 V140"/>
      <path d="M230 108 V120 H265 V140"/>
    </g>
    <g font-size="11" fill="currentColor" opacity="0.85">
      <text x="0" y="200">• ordered, balanced</text>
      <text x="0" y="216">• fast point + range</text>
      <text x="0" y="232">• read-friendly</text>
      <text x="0" y="248">• writes rebalance tree</text>
    </g>
  </g>
  <g transform="translate(340,40)">
    <text x="145" y="0" text-anchor="middle" fill="currentColor" font-weight="600">LSM-tree (Cassandra, RocksDB)</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="100" y="20" width="90" height="28" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="40" y="70" width="60" height="22" rx="4"/>
      <rect x="110" y="70" width="60" height="22" rx="4"/>
      <rect x="180" y="70" width="60" height="22" rx="4"/>
      <rect x="60" y="110" width="80" height="22" rx="4"/>
      <rect x="160" y="110" width="80" height="22" rx="4"/>
      <rect x="80" y="150" width="140" height="22" rx="4"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="145" y="38">memtable (RAM)</text>
      <text x="70" y="85">L0 SST</text>
      <text x="140" y="85">L0 SST</text>
      <text x="210" y="85">L0 SST</text>
      <text x="100" y="125">L1 SST</text>
      <text x="200" y="125">L1 SST</text>
      <text x="150" y="165">L2 SST</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M145 48 V70" stroke-dasharray="3 3"/>
      <path d="M145 92 V110" stroke-dasharray="3 3"/>
      <path d="M150 132 V150" stroke-dasharray="3 3"/>
    </g>
    <g font-size="11" fill="currentColor" opacity="0.85">
      <text x="0" y="200">• append-only writes</text>
      <text x="0" y="216">• background compaction</text>
      <text x="0" y="232">• write-friendly</text>
      <text x="0" y="248">• reads check many files</text>
    </g>
  </g>
</svg>

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

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 220" role="img" aria-label="Composite index on (country, city, zip) and which queries it serves" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Composite index on (country, city, zip)</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="40" y="50" width="120" height="40" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="180" y="50" width="120" height="40" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="320" y="50" width="120" height="40" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
  </g>
  <g fill="currentColor" text-anchor="middle">
    <text x="100" y="74" font-weight="700">country</text>
    <text x="240" y="74" font-weight="700">city</text>
    <text x="380" y="74" font-weight="700">zip</text>
  </g>
  <g font-size="12" fill="currentColor">
    <text x="40" y="130">✓ WHERE country = 'US'</text>
    <text x="40" y="152">✓ WHERE country = 'US' AND city = 'SF'</text>
    <text x="40" y="174">✓ WHERE country = 'US' AND city = 'SF' AND zip = '94107'</text>
    <text x="40" y="200" fill="var(--sl-color-accent,#3b82f6)">✗ WHERE city = 'SF'  — leading column missing → index unused</text>
  </g>
</svg>

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
