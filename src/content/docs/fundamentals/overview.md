---
title: Overview
description: What a system design interview is, what interviewers grade on, and how to use this site to prepare.
---

A system design interview asks you to design a large-scale software system — usually something real-world like a URL shortener, a chat app, a news feed, or a ride-sharing platform — in 45 to 60 minutes. There is no compiler, no test suite, and almost never a single correct answer. What is being measured is how you *think*: how you scope a problem, how you reason about trade-offs, and how you defend the decisions you make.

This page is the starting point. By the end of it you should know what to expect, what good answers look like, and how to use the rest of the site.

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

The sidebar is organized in the order you should learn the material:

- **Fundamentals** — read these first. The [Interview Framework](/fundamentals/interview-framework/) is the spine of every answer; [Requirements Gathering](/fundamentals/requirements/) and [Estimation](/fundamentals/estimation/) are the muscle. [CAP Theorem](/fundamentals/cap-theorem/) names the trade-offs you'll keep returning to.
- **Networking & Delivery, Data & Storage** — these are the components. You'll combine them in nearly every design.
- **Scalability & Reliability** — these are the techniques that turn a working design into one that survives 10x growth and partial failures.
- **Architecture Patterns** — vocabulary for talking about how the pieces fit together.

Each page is short on purpose. The aim is to give you the words and the trade-offs, not to be an encyclopedia. Read the page, then close it and try to explain the concept out loud in your own words. If you cannot, read it again.

## A study plan that works

If you have **four weeks**, spend one week per major section, ending the last week on practice problems and mock interviews. If you have **one week**, read every page in Fundamentals end-to-end on day one, then sprint through the rest in topical chunks. If you have **two days**, skim everything once, then re-read [Load Balancers](/networking/load-balancers/), [Caching](/data/caching/), [SQL vs NoSQL](/data/sql-vs-nosql/), and [Sharding](/data/sharding/) — those four topics show up in almost every design.

Whatever the timeline, pair reading with reps. System design is a skill, and skills don't grow by reading alone.
