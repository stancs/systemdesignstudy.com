---
title: Overview
description: What a system design interview is, what interviewers grade on, and how to use this site to prepare.
---

A system design interview asks you to design a large-scale software system — usually something real-world like a URL shortener, a chat app, a news feed, or a ride-sharing platform — in 45 to 60 minutes. There is no compiler, no test suite, and almost never a single correct answer. What is being measured is how you *think*: how you scope a problem, how you reason about trade-offs, and how you defend the decisions you make.

This page is the starting point. By the end of it you should know what to expect, what good answers look like, and how to use the rest of the site.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 220" role="img" aria-label="The four axes interviewers grade on" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:14px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600" font-size="14">What the interview actually measures</text>
  <g transform="translate(20,50)">
    <rect width="140" height="140" rx="10" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2"/>
    <text x="70" y="32" text-anchor="middle" fill="currentColor" font-weight="600">Framing</text>
    <text x="70" y="70" text-anchor="middle" fill="currentColor" font-size="12">Functional vs</text>
    <text x="70" y="86" text-anchor="middle" fill="currentColor" font-size="12">non-functional</text>
    <text x="70" y="110" text-anchor="middle" fill="currentColor" font-size="12">Scope &amp; scale</text>
  </g>
  <g transform="translate(180,50)">
    <rect width="140" height="140" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="70" y="32" text-anchor="middle" fill="currentColor" font-weight="600">Components</text>
    <text x="70" y="70" text-anchor="middle" fill="currentColor" font-size="12">LB, cache, queue,</text>
    <text x="70" y="86" text-anchor="middle" fill="currentColor" font-size="12">DB, replication</text>
    <text x="70" y="110" text-anchor="middle" fill="currentColor" font-size="12">Pick &amp; justify</text>
  </g>
  <g transform="translate(340,50)">
    <rect width="140" height="140" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="70" y="32" text-anchor="middle" fill="currentColor" font-weight="600">Trade-offs</text>
    <text x="70" y="70" text-anchor="middle" fill="currentColor" font-size="12">Consistency,</text>
    <text x="70" y="86" text-anchor="middle" fill="currentColor" font-size="12">latency, cost</text>
    <text x="70" y="110" text-anchor="middle" fill="currentColor" font-size="12">Defend choices</text>
  </g>
  <g transform="translate(500,50)">
    <rect width="120" height="140" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="60" y="32" text-anchor="middle" fill="currentColor" font-weight="600">Communication</text>
    <text x="60" y="70" text-anchor="middle" fill="currentColor" font-size="12">Drive the</text>
    <text x="60" y="86" text-anchor="middle" fill="currentColor" font-size="12">conversation</text>
    <text x="60" y="110" text-anchor="middle" fill="currentColor" font-size="12">Take feedback</text>
  </g>
</svg>

## What interviewers are actually grading

In most loops, the rubric collapses to four things:

**Problem framing.** Do you separate functional from non-functional requirements? Do you ask about scale, latency, and consistency *before* you start drawing boxes? A candidate who immediately starts sketching microservices for a problem that fits on a single Postgres instance is a red flag.

**Component knowledge.** Do you know what a load balancer, cache, queue, and replicated database actually do? Can you name a reasonable choice (e.g., "Redis for the cache, Kafka for the event stream") and explain *why* it fits?

**Trade-off reasoning.** Every meaningful design decision has a cost. Strong candidates surface those trade-offs unprompted: "I'm picking eventual consistency here because reads dominate writes 100:1, and a 200ms staleness window is acceptable for this product."

**Communication.** Can you drive a conversation, take a hint, and revise gracefully when the interviewer pushes back? An interviewer is trying to imagine working with you for the next several years. Defensive or scattered candidates lose the loop even when their designs are technically fine.

## What a good answer looks like

A good answer is a story, not a diagram. It moves in a deliberate order:

1. Clarify requirements and scope.
2. Estimate scale (QPS, storage, bandwidth).
3. Define the public API.
4. Sketch the high-level architecture.
5. Design the data model.
6. Drill into one or two deep dives that the interviewer steers you toward.
7. Discuss bottlenecks, failure modes, and what you would change at 10x scale.

Most candidates rush steps 1 and 2 because they feel like they are not "real" work. They are. An interviewer who hears you confidently say *"reads dominate writes about 100:1, peak QPS is around 50k, and we'll need roughly 2 TB of metadata storage in year one"* has already mentally upgraded your performance.

## How to use this site

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 260" role="img" aria-label="The five study tracks of this site" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:14px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">The five study tracks</text>
  <g transform="translate(250,40)">
    <rect width="140" height="50" rx="10" fill="currentColor" fill-opacity="0.08" stroke="currentColor" stroke-width="2"/>
    <text x="70" y="30" text-anchor="middle" fill="currentColor" font-weight="700">Fundamentals</text>
  </g>
  <g transform="translate(20,140)">
    <rect width="140" height="54" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="70" y="23" text-anchor="middle" fill="currentColor" font-weight="700">Networking</text>
    <text x="70" y="40" text-anchor="middle" fill="currentColor" font-weight="700">&amp; Delivery</text>
  </g>
  <g transform="translate(180,140)">
    <rect width="140" height="54" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="70" y="23" text-anchor="middle" fill="currentColor" font-weight="700">Data</text>
    <text x="70" y="40" text-anchor="middle" fill="currentColor" font-weight="700">&amp; Storage</text>
  </g>
  <g transform="translate(340,140)">
    <rect width="140" height="54" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="70" y="23" text-anchor="middle" fill="currentColor" font-weight="700">Scalability</text>
    <text x="70" y="40" text-anchor="middle" fill="currentColor" font-weight="700">&amp; Reliability</text>
  </g>
  <g transform="translate(500,140)">
    <rect width="120" height="54" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="60" y="23" text-anchor="middle" fill="currentColor" font-weight="700">Architecture</text>
    <text x="60" y="40" text-anchor="middle" fill="currentColor" font-weight="700">Patterns</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none">
    <path d="M320 90 V120 H90 V140"/>
    <path d="M320 90 V120 H250 V140"/>
    <path d="M320 90 V120 H410 V140"/>
    <path d="M320 90 V120 H560 V140"/>
  </g>
  <text x="320" y="228" text-anchor="middle" fill="currentColor" font-size="12" opacity="0.8">Read top to bottom: foundations first, components next, scaling and patterns last.</text>
</svg>

The sidebar is organized in the order you should learn the material:

- **Fundamentals** — read these first. The [Interview Framework](/fundamentals/interview-framework/) is the spine of every answer; [Requirements Gathering](/fundamentals/requirements/) and [Estimation](/fundamentals/estimation/) are the muscle. [CAP Theorem](/fundamentals/cap-theorem/) names the trade-offs you'll keep returning to.
- **Networking & Delivery, Data & Storage** — these are the components. You'll combine them in nearly every design.
- **Scalability & Reliability** — these are the techniques that turn a working design into one that survives 10x growth and partial failures.
- **Architecture Patterns** — vocabulary for talking about how the pieces fit together.

Each page is short on purpose. The aim is to give you the words and the trade-offs, not to be an encyclopedia. Read the page, then close it and try to explain the concept out loud in your own words. If you cannot, read it again.

## A study plan that works

If you have **four weeks**, spend one week per major section, ending the last week on practice problems and mock interviews. If you have **one week**, read every page in Fundamentals end-to-end on day one, then sprint through the rest in topical chunks. If you have **two days**, skim everything once, then re-read [Load Balancers](/networking/load-balancers/), [Caching](/data/caching/), [SQL vs NoSQL](/data/sql-vs-nosql/), and [Sharding](/data/sharding/) — those four topics show up in almost every design.

Whatever the timeline, pair reading with reps. System design is a skill, and skills don't grow by reading alone.
