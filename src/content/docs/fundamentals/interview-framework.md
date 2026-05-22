---
title: Interview Framework
description: A repeatable seven-step framework for working through any system design interview, with timings, prompts, and common pitfalls.
---

The single most important habit you can build is a **repeatable structure** for the interview itself. The framework below works for almost any prompt — URL shortener, news feed, ride-sharing, video streaming, real-time chat. Internalize it until it feels automatic, and you free up your cognitive budget for the *actual* design.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 680 200" role="img" aria-label="The seven steps of a system design interview" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="340" y="22" text-anchor="middle" fill="currentColor" font-weight="600">The seven-step framework</text>
  <line x1="40" y1="100" x2="640" y2="100" stroke="currentColor" stroke-width="2"/>
  <g font-size="11">
    <g transform="translate(40,100)">
      <circle r="14" fill="var(--sl-color-accent,#3b82f6)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <text y="4" text-anchor="middle" fill="var(--sl-color-white,#fff)" font-weight="700">1</text>
      <text y="-25" text-anchor="middle" fill="currentColor" font-weight="600">Clarify</text>
      <text y="35" text-anchor="middle" fill="currentColor" opacity="0.8">5 min</text>
    </g>
    <g transform="translate(140,100)">
      <circle r="14" fill="none" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">2</text>
      <text y="-25" text-anchor="middle" fill="currentColor" font-weight="600">Estimate</text>
      <text y="35" text-anchor="middle" fill="currentColor" opacity="0.8">3 min</text>
    </g>
    <g transform="translate(240,100)">
      <circle r="14" fill="none" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">3</text>
      <text y="-25" text-anchor="middle" fill="currentColor" font-weight="600">API</text>
      <text y="35" text-anchor="middle" fill="currentColor" opacity="0.8">3 min</text>
    </g>
    <g transform="translate(340,100)">
      <circle r="14" fill="none" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">4</text>
      <text y="-25" text-anchor="middle" fill="currentColor" font-weight="600">Architecture</text>
      <text y="35" text-anchor="middle" fill="currentColor" opacity="0.8">8 min</text>
    </g>
    <g transform="translate(440,100)">
      <circle r="14" fill="none" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">5</text>
      <text y="-25" text-anchor="middle" fill="currentColor" font-weight="600">Data model</text>
      <text y="35" text-anchor="middle" fill="currentColor" opacity="0.8">5 min</text>
    </g>
    <g transform="translate(540,100)">
      <circle r="14" fill="var(--sl-color-accent,#3b82f6)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <text y="4" text-anchor="middle" fill="var(--sl-color-white,#fff)" font-weight="700">6</text>
      <text y="-25" text-anchor="middle" fill="currentColor" font-weight="600">Deep dives</text>
      <text y="35" text-anchor="middle" fill="currentColor" opacity="0.8">15 min</text>
    </g>
    <g transform="translate(640,100)">
      <circle r="14" fill="none" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">7</text>
      <text y="-25" text-anchor="middle" fill="currentColor" font-weight="600">Wrap up</text>
      <text y="35" text-anchor="middle" fill="currentColor" opacity="0.8">5 min</text>
    </g>
  </g>
  <text x="340" y="180" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.7">Highlighted steps are where points are most often won.</text>
</svg>

## The seven steps

For a 45-minute interview, target these timings:

| Step | Time | Goal |
|------|------|------|
| 1. Clarify requirements | 5 min | Pin down what to build and what *not* to build |
| 2. Back-of-the-envelope estimation | 3 min | Establish scale (QPS, storage, bandwidth) |
| 3. API design | 3 min | Define the public interface |
| 4. High-level architecture | 8 min | Draw the boxes and arrows |
| 5. Data model | 5 min | Schema, keys, access patterns |
| 6. Deep dives | 15 min | Drill into bottlenecks the interviewer steers you toward |
| 7. Wrap up | 5 min | Trade-offs, failure modes, what changes at 10x |

You will almost never hit these times perfectly. Treat them as guardrails: if you are 15 minutes in and haven't drawn a box, slow down on requirements; if you are 30 minutes in and still on the API, accelerate.

## Step 1 — Clarify requirements

Split into **functional** and **non-functional**:

- **Functional**: what does the system *do*? "Users post short messages, follow other users, see a feed of posts from people they follow."
- **Non-functional**: how does it have to *behave*? Scale (DAU, QPS), latency (p99 < 200ms), availability (99.9% vs 99.99%), consistency expectations, regulatory or geographic constraints.

Always ask the interviewer; do not assume. Good questions: *"How many daily active users? Read-heavy or write-heavy? Should the feed be strictly chronological or ranked? Do we need to support media uploads?"*

Pitfall: trying to handle every possible feature. Pick the **core** functionality and explicitly defer the rest ("I'll skip notifications and search for now — happy to come back to them if there's time").

## Step 2 — Estimation

Quick math, out loud. The full toolbox is in [Back-of-the-Envelope Estimation](/fundamentals/estimation/), but in the moment you need three numbers:

- **QPS** (queries per second), broken down read vs write.
- **Storage** per year (rows × size × growth).
- **Bandwidth** (especially if media or video is involved).

The numbers don't need to be exact — they need to be *defensible*. "100M DAU, each posts on average twice a day, so ~2.3k writes/sec; reads are roughly 100x that at 230k/sec" is plenty.

## Step 3 — API design

Sketch 3–5 endpoints that cover the core flows. For each, list the method, path, key parameters, and return type. Keep it tight:

```
POST /posts          body: { content }         -> { post_id }
GET  /feed?cursor=…  -> { posts: [...], next_cursor }
POST /follow/:user   -> 204
```

This is also where you decide between **REST, gRPC, or GraphQL** — usually REST unless you have a reason. The [Communication Protocols](/networking/communication-protocols/) page goes deeper.

## Step 4 — High-level architecture

Now you draw boxes. A reasonable starting template:

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 360" role="img" aria-label="Reference high-level architecture used at step 4" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="270" y="10" width="100" height="40" rx="8"/>
    <rect x="270" y="80" width="100" height="40" rx="8"/>
    <rect x="240" y="150" width="160" height="40" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="40" y="240" width="120" height="50" rx="8"/>
    <rect x="180" y="240" width="120" height="50" rx="8"/>
    <rect x="320" y="240" width="120" height="50" rx="8"/>
    <rect x="460" y="240" width="140" height="50" rx="8"/>
    <rect x="380" y="320" width="120" height="30" rx="8"/>
  </g>
  <g fill="currentColor" text-anchor="middle">
    <text x="320" y="35">Client</text>
    <text x="320" y="105">Load Balancer</text>
    <text x="320" y="175" font-weight="600">App Servers</text>
    <text x="100" y="270">Cache</text>
    <text x="240" y="270">Database</text>
    <text x="380" y="270">Queue</text>
    <text x="530" y="270">Object Store</text>
    <text x="440" y="340">Workers</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none">
    <path d="M320 50 V80"/>
    <path d="M320 120 V150"/>
    <path d="M320 190 V210 H100 V240"/>
    <path d="M320 190 V210 H240 V240"/>
    <path d="M320 210 H380 V240"/>
    <path d="M320 190 V210 H530 V240"/>
    <path d="M380 290 V305 H440 V320"/>
  </g>
  <g fill="currentColor">
    <polygon points="316,82 320,76 324,82"/>
    <polygon points="316,152 320,146 324,152"/>
  </g>
</svg>


Walk through a write and a read end-to-end. Name the technology you'd pick at each box and one reason ("Postgres because we need transactions on follow relationships; Redis for the timeline cache because it supports sorted sets natively").

Pitfall: prematurely sharding, queueing, or microservicing. Start simple, then justify each piece of complexity by pointing back to your estimation numbers.

## Step 5 — Data model

Show the key tables or document shapes, the primary keys, and the most important indexes. Tie each one to an access pattern from step 1:

> *"`posts` is sharded by `user_id` because the dominant read is 'last N posts by a given user'. The feed cache is keyed by viewer_id, populated via fan-out-on-write — see [Sharding](/data/sharding/) for why I'm choosing that over fan-out-on-read."*

## Step 6 — Deep dives

This is where points are won or lost. The interviewer will pick one or two areas and push: *"How do you handle a celebrity user with 50M followers?" "What if the cache goes down?" "How do you prevent duplicate posts on retry?"*

Treat each deep dive as a mini design exercise:

1. Restate the problem.
2. Identify two or three approaches.
3. Pick one and explain the trade-off.
4. Mention failure modes.

Common deep-dive topics: hot keys, the thundering herd on cache rebuild, exactly-once semantics in queues, schema migrations under load, multi-region consistency.

## Step 7 — Wrap up

Spend the last few minutes on what you didn't have time for:

- **Bottlenecks** at the current design's limits.
- **What changes at 10x** the scale you estimated.
- **Failure modes** — what happens when component X dies?
- **Open questions** you'd want to investigate further.

This shows self-awareness. Interviewers love candidates who can finish with *"the part of this design I'm least confident about is X, and here's how I would validate it."*

## The biggest mistakes to avoid

- **Diving into solutions before scoping.** Spend the time on requirements. You'll move faster afterward.
- **Drawing without narrating.** The whiteboard is a prop; the story is what gets graded.
- **Refusing to commit.** "It depends" is a death knell when it isn't followed by "...and here's the trade-off and what I'd pick."
- **Fighting feedback.** Interviewers nudge you for a reason. Accept the nudge, integrate it, move on.
- **Skipping the wrap-up.** A confident "here's what I'd do differently at 10x" is worth more than another five minutes of drawing.

Practice this framework on three or four problems and it stops feeling like a script. That is when interviews start feeling like a conversation instead of a performance.
