---
title: Event-Driven Architecture
description: Events vs commands, pub/sub vs queues, event sourcing, CQRS, sagas — and the trade-offs every event-driven design has to defend.
---

Event-driven architecture (EDA) is what you get when services communicate by **publishing facts that already happened** rather than **calling each other to ask for things**. It is the natural shape of decoupled, scalable, asynchronous systems — and it is also the source of an enormous amount of accidental complexity in systems that adopt it without justification.

This page is the vocabulary you need to talk about EDA the way a senior engineer does.

## Events vs commands

A useful distinction that gets confused regularly.

A **command** is an instruction: "Send this email." It expresses intent. The sender wants something done; some service must accept it. If the service is unavailable, the command fails or queues.

An **event** is a notification of fact: "An order was placed." It expresses something that already happened. The publisher doesn't know or care who is listening. Subscribers react in their own time.

Both can flow through queues. The architectural difference is in **coupling**: commands are point-to-point and the sender knows who the receiver is; events are broadcast and the publisher doesn't.

A general guideline: **commands for "make this happen," events for "this happened, react if you care."** The combination shows up in real systems: a service receives a command, processes it, and emits an event when done.

## Pub/sub, queues, and streams

EDA needs a transport. Three flavors that show up:

- **Queues** — one-to-one delivery, one consumer wins. Use for commands and work distribution.
- **Pub/sub** — one-to-many broadcast, each subscriber gets its own copy. Use for events.
- **Streams** — persistent, replayable, ordered logs (Kafka, Pulsar). Use for events where late subscribers, replay, or auditability matter.

The lines blur in modern systems. Kafka can do all three, depending on consumer-group configuration. See [Message Queues & Streams](/data/message-queues/) for the underlying details.

## Why event-driven at all?

Real reasons to reach for EDA:

- **Decoupling.** Service A doesn't need to know who depends on its outputs. Adding a new consumer doesn't require changing A.
- **Smoothing spikes.** Bursty producers and steady consumers (or vice versa) can coexist without falling over.
- **Fan-out.** One business event ("user signed up") triggers ten things — welcome email, analytics, CRM, recommendations warmup, onboarding tutorial. Each is its own subscriber, evolves independently, can fail independently.
- **Replayability.** A persistent log lets a new service "catch up" on history without the publishers doing anything special.
- **Asynchronous reliability.** The work happens reliably even if the consumer is briefly down, slow, or fully unavailable.

Bad reasons:

- "It's loosely coupled." Yes — loosely coupled in a way that is much harder to debug.
- "We want event sourcing." Event sourcing is a separate pattern (below); don't confuse it with "we use events."

## Patterns built on events

### Event notification

The simplest pattern. Service A emits an event when something happens. Other services react. Each subscriber re-fetches data from A if it needs more detail.

- **Pros:** Minimal coupling, small messages, A's data stays inside A.
- **Cons:** Subscribers must hit A's API to enrich, which couples them again. Replay is harder because you need the historical state at the time of the event.

### Event-carried state transfer

The event carries enough state for subscribers to do their work without calling back. "Order placed" carries the full order payload.

- **Pros:** Subscribers can operate offline; no chatty callbacks; replay is meaningful.
- **Cons:** Larger messages; data is duplicated; schema evolution must be backward-compatible.

This is the workhorse pattern in modern EDA. Use it as the default.

### Event sourcing

The event log itself **is** the source of truth. Application state is a *projection* — the result of replaying every event from the beginning (or from a snapshot).

- **Pros:** Full auditability of every change. New projections (search indexes, read models, analytics) can be built from history without disturbing the source.
- **Cons:** Significant complexity. Event schemas live forever (you can never delete an event and still replay correctly). Queries against the current state require projections to be maintained.

Reach for event sourcing in domains where audit is critical (finance, healthcare) or where you genuinely benefit from rebuilding state at will. Don't reach for it because it sounds elegant. Most teams that adopt it without those needs regret it.

### CQRS (Command Query Responsibility Segregation)

Separate the write model from the read model. Commands change state through one pipeline; queries read from a different pipeline optimized for read patterns. Events flow from the write side to update read models.

- **Pros:** Read and write are independently scalable and optimizable. Multiple read models (search index, materialized view, analytics cube) can coexist.
- **Cons:** Two models to maintain. Read models lag the write model (eventual consistency). More moving parts.

CQRS and event sourcing pair famously well, but you can do either independently. Use CQRS when the read patterns are dramatically different from the write patterns (different shapes, different latency budgets, different access frequencies).

### Sagas: transactions across services

The hard problem in event-driven systems: how do you do a multi-step business operation that crosses services, atomically, when there is no distributed transaction?

Answer: you don't. You use a **saga** — a sequence of local transactions, each of which emits an event that triggers the next, and each step has a **compensating action** that undoes its work if a later step fails.

Two flavors:

- **Choreography.** Each service listens for the relevant event and acts; no central coordinator. Simple, but the business process is implicit in the event flow.
- **Orchestration.** A central coordinator (saga manager) sends commands to each service and tracks the state. Easier to reason about; the coordinator is another service to operate.

Both are right; the orchestration version is easier to debug and easier to evolve, at the cost of one more component. Most "real" sagas in production are orchestrated.

## What you have to handle (no exceptions)

EDA hands you a bunch of new problems for free:

**Idempotency.** Every event will be delivered at least once, sometimes twice. Consumers must handle duplicates gracefully — usually by tracking event IDs.

**Ordering.** Global ordering is rarely worth the cost; per-key ordering is usually what you actually want. Pick a partition key and call it out: *"events are per-user-ordered, no global order."*

**Schema evolution.** Once an event exists, it exists forever (especially with event sourcing). Use a schema registry, version every event, only make backwards-compatible changes (additive fields with defaults).

**Eventual consistency.** Subscribers see updates seconds, minutes, sometimes hours after the producer. The product has to be designed for this — UI must show pending states, expose "you'll see this in a moment" semantics, or simply tolerate stale reads.

**Observability.** Without distributed tracing across events, you cannot debug anything. Propagate trace context through every event. Build dashboards that show end-to-end pipeline health, not just per-service health.

**Dead-letter handling.** When an event can't be processed, it has to go somewhere humans can see. Always. With alerts.

## When event-driven is the right answer

Reach for EDA when:

- Multiple consumers need to react to the same business fact.
- Producers and consumers operate at very different speeds.
- You need durable, replayable history.
- You can tolerate eventual consistency on the data flow.
- You have the observability investment to operate it.

Avoid EDA when:

- You need a synchronous response with the result. Use a request-response API.
- The flow is fundamentally point-to-point.
- The team is small and the operational complexity isn't justified.

## What to say in an interview

A clean event-driven paragraph:

> *"Order placement is synchronous: the API validates and writes to the orders database, then publishes an `OrderPlaced` event to Kafka, partitioned by user_id for per-user ordering. Subscribers are independent — inventory reservation, payment processing, fulfillment, and analytics. Each is idempotent on order_id, retries with exponential backoff, and routes to a DLQ after three failures. Multi-step flows like 'place order → reserve inventory → charge card → ship' are coordinated by an orchestrated saga so we can compensate cleanly if any step fails. Schemas are versioned and registered, and we propagate trace context through every event so we can debug end-to-end."_*

Seven specific decisions in one paragraph. That is what mature event-driven design looks like, and it's the bar for the architecture-patterns deep dive.

## Common pitfalls

**Distributed monolith via events.** Twelve services all required to make one request work, just over Kafka instead of HTTP. Worse than the monolith you started with.

**No replay strategy.** New consumer needs historical data; the only option is to write a custom backfill script. Plan replay from day one.

**Synchronous expectations on asynchronous flows.** A user clicks "Submit" and the next page assumes the work has completed. With events, it has not. Show progress.

**Schema chaos.** No registry, no versioning, every team invents fields ad hoc. The bus becomes an integration bog. A schema registry is non-optional past trivial scale.

**Untraced events.** Without distributed tracing across event boundaries, debugging is archaeology. Propagate trace context.

Event-driven systems pay back enormous decoupling dividends — *if* you've done the engineering to operate them well. If not, they punish you. Pick deliberately.
