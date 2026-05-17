---
title: Requirements Gathering
description: How to scope a system design problem the way senior engineers do — separating functional from non-functional requirements and locking in the trade-offs.
---

A system design interview is won or lost in the first ten minutes. Candidates who launch into solutions without clarifying the problem end up designing something the interviewer didn't ask for, then have to backtrack when the constraints they assumed turn out to be wrong. Candidates who scope carefully look senior even when their downstream design is average.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 280" role="img" aria-label="Functional and non-functional requirements side by side" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.4 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Two layers of requirements</text>
  <g transform="translate(20,45)">
    <rect width="290" height="215" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="145" y="28" text-anchor="middle" fill="currentColor" font-weight="700">Functional</text>
    <text x="145" y="46" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">what the system does</text>
    <g fill="currentColor" font-size="12">
      <text x="20" y="80">• Users post messages</text>
      <text x="20" y="105">• Users follow other users</text>
      <text x="20" y="130">• Users see a feed</text>
      <text x="20" y="155">• Users upload media</text>
      <text x="20" y="180">• Users get notifications</text>
    </g>
    <text x="145" y="200" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.7">User-visible features</text>
  </g>
  <g transform="translate(330,45)">
    <rect width="290" height="215" rx="10" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2"/>
    <text x="145" y="28" text-anchor="middle" fill="currentColor" font-weight="700">Non-functional</text>
    <text x="145" y="46" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">how it behaves</text>
    <g fill="currentColor" font-size="12">
      <text x="20" y="80">• Scale: 100M DAU, 50k QPS</text>
      <text x="20" y="105">• Latency: p99 &lt; 200 ms</text>
      <text x="20" y="130">• Availability: 99.95%</text>
      <text x="20" y="155">• Consistency: read-your-writes</text>
      <text x="20" y="180">• Cost &amp; regulatory limits</text>
    </g>
    <text x="145" y="200" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">Drives every architectural choice</text>
  </g>
</svg>

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

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 200" role="img" aria-label="Consistency spectrum from strong to eventual" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">The consistency spectrum</text>
  <line x1="40" y1="100" x2="600" y2="100" stroke="currentColor" stroke-width="2"/>
  <g stroke="currentColor" stroke-width="2" fill="currentColor">
    <line x1="80" y1="93" x2="80" y2="107"/>
    <line x1="270" y1="93" x2="270" y2="107"/>
    <line x1="450" y1="93" x2="450" y2="107"/>
    <line x1="580" y1="93" x2="580" y2="107"/>
  </g>
  <g text-anchor="middle" fill="currentColor">
    <text x="80" y="80" font-weight="600">Strong</text>
    <text x="80" y="130" font-size="12">money,</text>
    <text x="80" y="146" font-size="12">inventory</text>
    <text x="270" y="80" font-weight="600">Read-your-writes</text>
    <text x="270" y="130" font-size="12">user-generated</text>
    <text x="270" y="146" font-size="12">content</text>
    <text x="450" y="80" font-weight="600">Causal</text>
    <text x="450" y="130" font-size="12">chat, collab</text>
    <text x="450" y="146" font-size="12">editing</text>
    <text x="580" y="80" font-weight="600">Eventual</text>
    <text x="580" y="130" font-size="12">view counts,</text>
    <text x="580" y="146" font-size="12">feeds</text>
  </g>
  <text x="40" y="180" fill="currentColor" font-size="11" opacity="0.7">slower, simpler to reason about</text>
  <text x="600" y="180" text-anchor="end" fill="currentColor" font-size="11" opacity="0.7">faster, harder to reason about</text>
</svg>

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
