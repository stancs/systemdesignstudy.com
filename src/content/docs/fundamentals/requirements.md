---
title: Requirements Gathering
description: How to scope a system design problem the way senior engineers do — separating functional from non-functional requirements and locking in the trade-offs.
---

A system design interview is won or lost in the first ten minutes. Candidates who launch into solutions without clarifying the problem end up designing something the interviewer didn't ask for, then have to backtrack when the constraints they assumed turn out to be wrong. Candidates who scope carefully look senior even when their downstream design is average.

## Functional vs non-functional

Every system has two layers of requirements.

**Functional requirements** describe *what the system does*. They are user-visible features:

- "Users can post messages up to 280 characters."
- "Users can follow other users."
- "Users can see a chronological feed of posts from people they follow."

**Non-functional requirements** describe *how the system behaves*. They are the constraints that drive every architectural decision downstream:

- Scale (DAU/MAU, peak QPS, storage growth).
- Latency (p50/p99 targets for reads and writes).
- Availability target (99.9%, 99.99%, 99.999%).
- Consistency expectations (strong, eventual, read-your-writes).
- Durability and recovery (RPO, RTO).
- Cost sensitivity.
- Geographic distribution, compliance, data residency.

Most candidates remember the functional part and skip the non-functional. That's backwards: the non-functional requirements are the ones that decide whether you reach for Postgres or DynamoDB, whether you need a queue, whether you shard, and how aggressively you cache.

## A starter list of questions to ask

Pick from this menu in the first few minutes. You don't need every answer — but the act of asking is signal:

**Scope and scale**

- Who are the users? Public consumer product or internal enterprise tool?
- How many daily/monthly active users?
- What's the read-to-write ratio?
- Are there obvious traffic peaks (e.g., evenings, live events)?
- What's the expected growth in the next 1–2 years?

**Performance**

- What p99 latency is acceptable for the main flows?
- Are any operations explicitly allowed to be slow (e.g., analytics, daily digests)?
- Is the system primarily synchronous or can we lean on async processing?

**Data**

- What is the typical record size? Largest record?
- How long do we keep data? Is anything time-bound?
- Are there strong-consistency operations (payments, balances, inventory)?

**Reliability**

- What is the availability target?
- What happens if the system is down — degraded service, hard outage, lost data?
- Is multi-region required, or is a single region acceptable?

**Constraints**

- Is there an existing tech stack we have to use?
- Are there budget or hardware constraints?
- Are there regulatory constraints (GDPR, HIPAA, PCI)?

You will not have time to ask all of these. Pick three or four that obviously matter for the prompt and move on.

## Pinning down scale

Numbers turn vague problems into concrete ones. Once you have rough estimates, every design choice becomes easier to defend. A useful pattern is to convert user-level numbers into system-level numbers:

> *"100M DAU. Average user opens the app 10 times per day and reads ~30 posts per session. That's 100M × 10 × 30 = 30B feed reads per day, or roughly 350k reads per second on average and probably 1M QPS at peak. Writes are smaller — each user posts maybe twice a day, so 200M writes/day or ~2.3k/sec."*

That single paragraph tells the interviewer (and you) which numbers actually matter. You no longer need to debate "should we cache the feed?" — at 1M reads/sec you obviously must.

## Choosing your consistency story

If you remember one thing from this page, remember: **explicitly name the consistency level for every read path.**

- **Strong consistency** — every read reflects the latest committed write. Required for money, inventory, uniqueness checks, ACLs.
- **Read-your-writes** — a user always sees their own writes immediately, but other users may lag briefly. Standard for social products.
- **Eventual consistency** — reads may be stale by seconds or minutes. Fine for view counts, feeds, search indexes, recommendations.

Most real systems are a *mix*: strong for some endpoints, eventual for others. Call that mix out loud during requirements. The [CAP Theorem](/fundamentals/cap-theorem/) page goes deeper.

## What to explicitly defer

In an interview, it is almost always a mistake to try to support every feature. Pick a core slice and say so:

> *"I'm going to focus on posting, following, and the home feed. I'll skip search, notifications, and content moderation — happy to come back to them if there's time at the end."*

This is one of the strongest signals you can send. It shows you can prioritize, you understand interview time pressure, and you respect the interviewer's time as much as your own.

## A reusable template

Here is a one-paragraph template you can write at the top of the whiteboard and fill in:

> **Goal:** _[1-sentence summary of the product]_
> **Core features:** _[3–5 bullets]_
> **Out of scope:** _[2–4 bullets explicitly deferred]_
> **Scale:** _[DAU, QPS read/write, storage/year]_
> **Latency:** _[p99 for reads / writes]_
> **Availability:** _[target, regions]_
> **Consistency:** _[strong where, eventual where]_

If you walk through that template in five minutes and the interviewer agrees, you have already done more useful work than half the candidates who get the same prompt.
