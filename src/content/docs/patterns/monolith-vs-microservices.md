---
title: Monolith vs Microservices
description: A senior take on the monolith/microservices debate — what each actually buys you, what each actually costs, and when to make the move.
---

The "should we use microservices?" question has been beaten to death and back, but it still comes up in interviews because the answer is genuinely interesting and reveals whether you've actually operated systems or just read about them.

The short version: **monoliths are usually the right starting point**, and microservices are a response to specific scaling pains (organizational and technical) that you should be able to name. Reaching for microservices first because "they scale" is the canonical junior mistake.

## What each thing actually is

**Monolith.** One deployable artifact (one container, one binary, one process tree) that contains the whole application. May be internally well-modularized or a swamp; the unit of deployment is one.

**Modular monolith.** A monolith with strong internal boundaries — well-defined modules with their own data, their own APIs, and explicit dependencies. Deploys as one unit but is *organized* like multiple services. Often the best architecture for teams up to ~30 engineers.

**Microservices.** Many independently deployable services, each owning a slice of the domain and (critically) its own data. Communication is over the network, usually HTTP or gRPC, sometimes via events.

**Service-oriented architecture (SOA).** The older, broader term — microservices are a stricter subset emphasizing small, independent, single-responsibility services.

These are points on a spectrum, not a binary.

## What monoliths are good at

The case is stronger than the trend press would have you believe:

- **Local reasoning.** Calls are in-process, so latency, error handling, and transactions are simple.
- **One deploy.** Atomically ship a change that touches three modules. No coordination, no version skew.
- **Cheap operations.** One process to deploy, monitor, debug. One database to back up.
- **Easy refactoring.** The compiler (or test suite) catches breaking changes immediately across module boundaries.
- **Performance.** No network hops, no serialization. A monolith is usually 10–100x faster for the same work than the equivalent microservices call graph.

A modular monolith with a clean internal architecture handles **most companies and most teams** for years. Many famously-microservice companies (Shopify, GitHub, Stack Overflow) ran enormous traffic on monoliths long before — and in some cases still — splitting them.

## What microservices are good at

The case is also real, just narrower than usually claimed:

- **Independent deploys.** Teams ship at their own pace. The payments team's deploy doesn't block the search team's. This is the single most underrated benefit.
- **Team scaling.** Above a certain headcount (50–100 engineers), a monolith's coordination cost outweighs its simplicity. Splitting by service mirrors splitting by team.
- **Independent scaling.** The image-processing service can run on 200 GPU instances while the auth service runs on 4 small CPUs.
- **Independent technology choices.** Service A in Go, service B in Python, service C wrapping a legacy Java library. Each team picks the right tool.
- **Fault isolation.** A bug in the recommendations service doesn't crash the checkout service.

Notice that several of these are **organizational** rather than technical. Conway's Law — your architecture mirrors your communication structure — applies powerfully here.

## What microservices actually cost

The costs are real and often surprise teams who only counted the benefits:

- **Network is unreliable.** Every in-process call becomes a remote call, with latency, partial failure, and serialization. You now need timeouts, retries, circuit breakers, idempotency.
- **Distributed transactions are very hard.** Multi-service writes either get sagas (compensating actions on failure) or accept eventual consistency. The cheap monolithic transaction is gone.
- **Operational complexity multiplies.** N services means N deploy pipelines, N sets of dashboards, N on-call rotations (or one very stretched rotation).
- **Debugging is harder.** A user-facing latency spike now requires distributed tracing to localize.
- **Testing is harder.** Integration tests span services and processes. Local development requires running half the company.
- **Versioning is permanent.** Every API contract between services becomes a long-lived interface that has to evolve compatibly.
- **Data ownership becomes a question.** Two services need the same data; do they call each other every time, replicate it, or share a database? (The right answer is almost never "share a database.")

A widely-quoted line, paraphrasing Martin Fowler: *"You must be this tall to ride microservices."* You need to have already invested in CI/CD, observability, and platform infrastructure before microservices stop being a tax and start being a leverage point.

## The pragmatic middle: modular monolith

For teams of ~5–30 engineers, the modular monolith is usually the best architecture in the world. The recipe:

- One deployable, but internally organized as modules with clear boundaries.
- Each module owns its data (its own schema/tables; no other module reads them directly).
- Inter-module calls go through explicit interfaces, even though they're in-process.
- A naming convention or tool (a "module manifest", language-level visibility) enforces the boundaries.

When (and only when) a module becomes load-isolated enough or team-owned enough to justify its own service, **extract it.** You already have the boundary; the extraction is mechanical.

This pattern is sometimes called the "modulith." Treat it as the senior default until proven otherwise.

## Communication patterns between services

Once you do have multiple services, two ways they talk:

**Synchronous (request-response).** Service A makes an HTTP/gRPC call to service B, waits for a response. Simple, easy to reason about. Couples availability — if B is down, A is down. Adds latency.

**Asynchronous (event-based).** Service A publishes an event; service B subscribes and reacts. Decoupled — A doesn't care whether B is healthy. Higher complexity (queues, idempotency, eventual consistency). See [Event-Driven Architecture](/patterns/event-driven/).

A useful rule: **synchronous for queries, asynchronous for things downstream services do in reaction.** The request-path stays simple; the fan-out work happens off the critical path.

## Data ownership

The single rule that distinguishes real microservices from a "distributed monolith":

> **Each service owns its data. No other service reads it directly.**

If service B needs data from service A, it either:

1. Calls A's API.
2. Subscribes to A's events and keeps a denormalized copy.
3. Reads from a derived store A publishes to (a CDC-fed search index, an analytics warehouse).

The wrong answer — and a depressingly common one in real systems — is for service B to read directly from service A's database. That couples the two services on schema in a way that defeats half the reason you split them in the first place.

## When to actually split

Defensible reasons:

- **A specific subsystem has fundamentally different scaling.** Image processing wants GPUs, auth wants CPUs.
- **A specific subsystem has a different deploy cadence.** The marketing pages ship 20 times a day; the payments code ships once a quarter.
- **Team boundaries are forcing coordination cost.** Two teams stomping on each other's deploys is a strong signal.
- **The blast radius of one subsystem's bugs needs to be contained.** Payments shouldn't be able to bring down browse.

Bad reasons:

- "Microservices scale better." Modulariths scale fine.
- "Everyone else uses them." Yes, and most of them regret it sometimes.
- "It's modern." Architectural decisions don't go on a resume.

## What to say in an interview

A senior monolith-vs-microservices answer:

> *"I'd start with a modular monolith — single deployable, but cleanly separated modules for users, orders, and payments, each owning its own tables. As the team grows past ~30 engineers or specific subsystems develop different scaling or deploy patterns, I'd extract them. The first candidate is usually the highest-traffic, most independent module — for an e-commerce system that's the catalog/browse path. Each service after that gets its own data store, communicates over gRPC for queries and Kafka for fan-out events, and follows our standard observability and resilience patterns."_*

Six concrete decisions, in order. That's how to talk about this without sounding like you're picking a side in a religious war.

## Common pitfalls

**Designing microservices in an interview "because the prompt is big."** Defend the choice with team size, deploy cadence, or scaling asymmetry. "It's a lot of users" is not enough — Stack Overflow ran on a monolith.

**Shared database.** Two services writing the same tables is a distributed monolith with extra latency.

**Synchronous chains.** A chain `A → B → C → D` has the availability of the weakest link multiplied. Break with events where possible.

**Premature splitting.** Every service you don't have is a service you don't have to operate.

The senior version of this topic is humility: pick the simplest architecture that solves the problem, and resist the urge to over-engineer for hypothetical future load.
