---
title: Interview Framework
description: A repeatable seven-step framework for working through any system design interview, with timings, prompts, and common pitfalls.
---

The single most important habit you can build is a **repeatable structure** for the interview itself. The framework below works for almost any prompt — URL shortener, news feed, ride-sharing, video streaming, real-time chat. Internalize it until it feels automatic, and you free up your cognitive budget for the *actual* design.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 680 168" role="img" aria-label="The seven steps of a system design interview" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="340" y="20" text-anchor="middle" fill="currentColor" font-weight="600">The seven-step framework</text>
  <line x1="50" y1="84" x2="630" y2="84" stroke="currentColor" stroke-width="2"/>
  <g font-size="11">
    <g transform="translate(50,84)">
      <circle r="15" fill="var(--sl-color-accent,#3b82f6)" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="var(--sl-color-white,#fff)" font-weight="700">1</text>
      <text y="-28" text-anchor="middle" fill="currentColor" font-weight="600">Clarify</text>
      <text y="36" text-anchor="middle" fill="currentColor" opacity="0.75">5 min</text>
    </g>
    <g transform="translate(147,84)">
      <circle r="15" fill="var(--sl-color-bg,#fff)" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">2</text>
      <text y="-28" text-anchor="middle" fill="currentColor" font-weight="600">Estimate</text>
      <text y="36" text-anchor="middle" fill="currentColor" opacity="0.75">3 min</text>
    </g>
    <g transform="translate(243,84)">
      <circle r="15" fill="var(--sl-color-bg,#fff)" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">3</text>
      <text y="-28" text-anchor="middle" fill="currentColor" font-weight="600">API</text>
      <text y="36" text-anchor="middle" fill="currentColor" opacity="0.75">3 min</text>
    </g>
    <g transform="translate(340,84)">
      <circle r="15" fill="var(--sl-color-bg,#fff)" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">4</text>
      <text y="-28" text-anchor="middle" fill="currentColor" font-weight="600">Architecture</text>
      <text y="36" text-anchor="middle" fill="currentColor" opacity="0.75">8 min</text>
    </g>
    <g transform="translate(437,84)">
      <circle r="15" fill="var(--sl-color-bg,#fff)" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">5</text>
      <text y="-28" text-anchor="middle" fill="currentColor" font-weight="600">Data model</text>
      <text y="36" text-anchor="middle" fill="currentColor" opacity="0.75">5 min</text>
    </g>
    <g transform="translate(533,84)">
      <circle r="15" fill="var(--sl-color-accent,#3b82f6)" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="var(--sl-color-white,#fff)" font-weight="700">6</text>
      <text y="-28" text-anchor="middle" fill="currentColor" font-weight="600">Deep dives</text>
      <text y="36" text-anchor="middle" fill="currentColor" opacity="0.75">15 min</text>
    </g>
    <g transform="translate(630,84)">
      <circle r="15" fill="var(--sl-color-bg,#fff)" stroke="currentColor" stroke-width="2"/>
      <text y="4" text-anchor="middle" fill="currentColor" font-weight="700">7</text>
      <text y="-28" text-anchor="middle" fill="currentColor" font-weight="600">Wrap up</text>
      <text y="36" text-anchor="middle" fill="currentColor" opacity="0.75">5 min</text>
    </g>
  </g>
  <text x="340" y="156" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.7">Steps 1 and 6 (highlighted) are where points are most often won.</text>
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

Sketch 3–5 endpoints that cover the core flows. For each, name the HTTP method, the path, the request parameters or body, and the response. Keep it tight — something like this for a simple social feed:

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 372" role="img" aria-label="Example REST API for a social feed with four endpoints" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">An example REST API — a social feed</text>

  <g>
    <rect x="14" y="40" width="612" height="74" rx="10" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.4"/>
    <rect x="28" y="56" width="78" height="28" rx="14" fill="#0F6E56"/>
    <text x="67" y="74" text-anchor="middle" fill="#ffffff" font-weight="700" font-size="11.5">POST</text>
    <text x="122" y="75" font-family="ui-monospace,SFMono-Regular,Menlo,monospace" font-weight="600" font-size="14" fill="currentColor">/posts</text>
    <text x="612" y="75" text-anchor="end" font-size="12" fill="currentColor" opacity="0.6">Create a new post</text>
    <text x="122" y="99" font-family="ui-monospace,SFMono-Regular,Menlo,monospace" font-size="11.5" fill="currentColor" opacity="0.8">body { content }   →   201 Created · { post_id }</text>
  </g>

  <g>
    <rect x="14" y="122" width="612" height="74" rx="10" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.4"/>
    <rect x="28" y="138" width="78" height="28" rx="14" fill="#185FA5"/>
    <text x="67" y="156" text-anchor="middle" fill="#ffffff" font-weight="700" font-size="11.5">GET</text>
    <text x="122" y="157" font-family="ui-monospace,SFMono-Regular,Menlo,monospace" font-weight="600" font-size="14" fill="currentColor">/feed</text>
    <text x="612" y="157" text-anchor="end" font-size="12" fill="currentColor" opacity="0.6">Fetch the home feed</text>
    <text x="122" y="181" font-family="ui-monospace,SFMono-Regular,Menlo,monospace" font-size="11.5" fill="currentColor" opacity="0.8">query ?cursor &amp; ?limit   →   200 OK · { posts[], next_cursor }</text>
  </g>

  <g>
    <rect x="14" y="204" width="612" height="74" rx="10" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.4"/>
    <rect x="28" y="220" width="78" height="28" rx="14" fill="#0F6E56"/>
    <text x="67" y="238" text-anchor="middle" fill="#ffffff" font-weight="700" font-size="11.5">POST</text>
    <text x="122" y="239" font-family="ui-monospace,SFMono-Regular,Menlo,monospace" font-weight="600" font-size="14" fill="currentColor">/posts/:id/likes</text>
    <text x="612" y="239" text-anchor="end" font-size="12" fill="currentColor" opacity="0.6">Like a post</text>
    <text x="122" y="263" font-family="ui-monospace,SFMono-Regular,Menlo,monospace" font-size="11.5" fill="currentColor" opacity="0.8">(no request body)   →   204 No Content</text>
  </g>

  <g>
    <rect x="14" y="286" width="612" height="74" rx="10" fill="none" stroke="currentColor" stroke-width="1.5" opacity="0.4"/>
    <rect x="28" y="302" width="78" height="28" rx="14" fill="#A32D2D"/>
    <text x="67" y="320" text-anchor="middle" fill="#ffffff" font-weight="700" font-size="11.5">DELETE</text>
    <text x="122" y="321" font-family="ui-monospace,SFMono-Regular,Menlo,monospace" font-weight="600" font-size="14" fill="currentColor">/posts/:id/likes</text>
    <text x="612" y="321" text-anchor="end" font-size="12" fill="currentColor" opacity="0.6">Remove a like</text>
    <text x="122" y="345" font-family="ui-monospace,SFMono-Regular,Menlo,monospace" font-size="11.5" fill="currentColor" opacity="0.8">(no request body)   →   204 No Content</text>
  </g>
</svg>

Notice the small design decisions worth saying out loud: the feed is **cursor-paginated** rather than offset-paginated (offsets drift when new posts arrive), and *like* and *unlike* are a `POST`/`DELETE` pair on the **same resource path** rather than two unrelated verbs. Small, consistent choices like these signal that you think in terms of REST resources, not ad-hoc RPC calls.

This is also where you decide between **REST, gRPC, or GraphQL** — usually REST unless you have a reason. The [Communication Protocols](/networking/communication-protocols/) page goes deeper.

## Step 4 — High-level architecture

Now you draw boxes. A reasonable starting template:

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 404" role="img" aria-label="Reference high-level architecture used at step 4" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <defs>
    <marker id="arch-ah" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="12" markerHeight="12" markerUnits="userSpaceOnUse" orient="auto">
      <path d="M0,1 L9,5 L0,9 z" fill="currentColor"/>
    </marker>
  </defs>

  <g stroke="currentColor" stroke-width="2" fill="none">
    <path d="M320 56 V92" marker-end="url(#arch-ah)"/>
    <path d="M320 136 V170" marker-end="url(#arch-ah)"/>
    <path d="M320 218 V242"/>
    <path d="M102 242 H532"/>
    <path d="M102 242 V264" marker-end="url(#arch-ah)"/>
    <path d="M246 242 V264" marker-end="url(#arch-ah)"/>
    <path d="M388 242 V264" marker-end="url(#arch-ah)"/>
    <path d="M532 242 V264" marker-end="url(#arch-ah)"/>
    <path d="M388 318 V350" marker-end="url(#arch-ah)"/>
  </g>

  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="262" y="14" width="116" height="42" rx="8"/>
    <rect x="262" y="94" width="116" height="42" rx="8"/>
    <rect x="230" y="170" width="180" height="48" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="44" y="266" width="116" height="52" rx="8"/>
    <rect x="182" y="266" width="128" height="52" rx="8"/>
    <rect x="332" y="266" width="112" height="52" rx="8"/>
    <rect x="462" y="266" width="140" height="52" rx="8"/>
    <rect x="328" y="350" width="120" height="46" rx="8"/>
  </g>

  <g fill="currentColor" text-anchor="middle">
    <text x="320" y="40">Client</text>
    <text x="320" y="120">Load Balancer</text>
    <text x="320" y="199" font-weight="600">App Servers</text>
    <text x="102" y="296">Cache</text>
    <text x="246" y="296">Database</text>
    <text x="388" y="296">Queue</text>
    <text x="532" y="296">Object Store</text>
    <text x="388" y="377">Workers</text>
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
