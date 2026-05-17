---
title: Back-of-the-Envelope Estimation
description: The numbers, units, and shortcuts you need to size systems on a whiteboard — QPS, storage, bandwidth, and the latency table every senior engineer carries in their head.
---

Back-of-the-envelope (BotE) estimation is the practice of doing rough capacity math out loud, on a whiteboard, in under three minutes. It is not about precision — it is about producing numbers good enough to drive the next design decision. An engineer who can confidently say *"that's about 50,000 writes per second and 2 TB of data per year"* sounds like an engineer who has built systems before.

## Why it matters

Estimation does two things in an interview:

1. **It justifies your architecture.** "I'm sharding the database" lands very differently after "we'll hit 200k writes/sec at peak."
2. **It establishes the scale axis.** Once you have numbers, you and the interviewer can debate *"what if it's 10x?"* — and 10x questions are where points live.

Without numbers, every design choice looks arbitrary. With numbers, every choice is defensible.

## Powers of two and ten

Memorize these two columns. They are the entire vocabulary of BotE math:

| Power of 10 | Name | Power of 2 | Approx |
|-------------|------|------------|--------|
| 10³ | thousand | 2¹⁰ | 1,024 |
| 10⁶ | million | 2²⁰ | ~1M |
| 10⁹ | billion | 2³⁰ | ~1B |
| 10¹² | trillion | 2⁴⁰ | ~1T |

You can swap 2¹⁰ for 10³ on a whiteboard and no one cares. That little trick is how senior engineers do arithmetic in their heads.

## Time and traffic

A few seconds-per-time-unit shortcuts that pay for themselves:

- 1 day ≈ 86,400 seconds — round to **100k seconds/day**.
- 1 month ≈ 2.5M seconds.
- 1 year ≈ 30M seconds.

So a system with 1 billion writes per day is doing 1B / 100k = **10,000 writes/sec on average**. Multiply by 3–5 for peak.

The rule of thumb for peak vs average: **peak is typically 2–10x average** depending on the product. Consumer products with daily rhythm spike higher than B2B.

## Data sizes that matter

You don't need to know every encoding, but you should know these:

- **char**: 1 byte (ASCII) / 2 bytes (UTF-16 in some langs).
- **integer**: 4 bytes.
- **long / timestamp**: 8 bytes.
- **UUID**: 16 bytes binary, 36 chars as text.
- **typical row**: 100–500 bytes for metadata, 1–10 KB if it includes a small blob.
- **image**: ~200 KB compressed (1080p photo); video is 1–10 MB per minute at modest quality.

A useful trick: when storing many small records, **the row overhead and indexes often double the raw size on disk**. A 100-byte row plus indexes is often 200–300 bytes of actual storage.

## The latency table

Every senior engineer has rough numbers in their head. You should too:

| Operation | Latency |
|-----------|---------|
| L1 cache reference | ~1 ns |
| L2 cache reference | ~5 ns |
| Main memory reference | ~100 ns |
| SSD random read (4 KB) | ~100 µs |
| Round trip in same datacenter | ~500 µs |
| HDD seek | ~10 ms |
| Round trip CA → Netherlands | ~150 ms |

Two implications you can quote almost verbatim:

- **Memory is ~100x faster than SSD**, which is ~100x faster than HDD, which is ~100x faster than a transcontinental round trip. Each step of the pyramid hides a 100x cost.
- **Network is the bottleneck.** If a request needs 5 sequential cross-region round trips, you've already spent 750 ms before you've done any real work.

## A worked example: a Twitter-scale feed

Walk through this out loud the first few times until it feels automatic.

**Users.** 300M MAU, 150M DAU. Average user opens the app 5 times a day.

**Writes.** Each user posts on average 0.5 times per day. That's 75M posts/day. Divided by 100k seconds, that's **750 posts/sec average**, peaking maybe 3x at ~2.5k/sec.

**Reads.** Each user reads ~50 posts per visit, 5 visits/day → 250 reads/user/day. 150M × 250 = 37.5B reads/day. That's **375k reads/sec average**, peaking ~1M reads/sec.

**Read:write ratio.** ~500:1. That immediately tells you: cache aggressively, denormalize for reads, and don't be afraid of fan-out-on-write costs.

**Storage.** Each post is ~300 bytes of metadata plus media references. 75M posts/day × 300 bytes = 22.5 GB/day → **~8 TB/year of post metadata**, before media, replication, or indexes. Apply 3x for replication = ~25 TB/year of disk.

**Bandwidth.** If average response payload is 50 KB (a feed page), and you do 1M reads/sec at peak, that's **50 GB/sec egress** at peak. That number is what justifies a CDN at the front.

In three minutes you've made the case for caching, fan-out-on-write, sharding, and a CDN — all by doing arithmetic.

## The three numbers you always need

If you forget everything else, derive these three for every interview:

1. **Peak QPS**, split read vs write.
2. **Storage per year**, with replication factor and indexes applied.
3. **Egress bandwidth**, especially if the response payload is large.

Get those three on the whiteboard early and the rest of the interview moves twice as fast.

## Common pitfalls

- **Confusing average and peak.** Always say which you mean. "10k QPS average, peak ~30k" is fine; "10k QPS" without qualification gets you challenged.
- **Forgetting replication and indexes.** Raw row size is rarely your real disk usage. 2–3x is a safe multiplier.
- **Ignoring bandwidth.** If the egress number is in the GB/sec range, mention a CDN. Interviewers notice when you don't.
- **Treating estimates as truth.** They are estimates. When the interviewer pushes ("what if it's 10x?"), redo the math live — don't defend the original number as if it were measured.

Practice this on three or four prompts and you'll find you can do the whole estimation step in under three minutes without thinking. That is the bar.
