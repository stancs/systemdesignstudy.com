---
title: Fault Tolerance & Redundancy
description: How to design systems that survive partial failure — redundancy, isolation, retries, timeouts, circuit breakers, bulkheads, and the SLO arithmetic interviewers expect you to know.
---

A distributed system is, by definition, a system where the failure of a computer you didn't know existed can render your own computer unusable (Lamport). The whole job of fault tolerance is making that *not* be true — designing so that the failure of any single component is invisible to the user.

This is the topic where interviewers most reward concrete failure-mode reasoning. "What happens when X breaks?" is the question that closes the loop, and you should always have an answer.

## Failure is not optional

A few numbers to keep in mind:

- A modern cloud VM has uptime around 99.95%. That's ~5 hours of downtime per year, *per instance*.
- Disks fail at roughly 1–2% AFR (annual failure rate). In a fleet of 10,000 disks, you can expect a failure every few days.
- Network partitions happen. Cross-zone packet loss is measured in real production systems.
- Entire datacenters and availability zones go down occasionally. Entire regions, rarely but it happens.

Design for these as certainties, not edge cases.

## The four building blocks

**Redundancy.** Multiple copies of every critical component (servers, databases, network paths, datacenters). One copy fails, the others carry the load. This is the *substrate* on which everything else is built.

**Isolation.** Failures stay contained to a small blast radius. A bug in one service doesn't bring down the rest; a hot tenant doesn't starve quiet ones; a poisonous message doesn't drain the whole worker pool.

**Detection.** Health checks, monitoring, alerting. You can't replace a broken instance you don't know is broken.

**Recovery.** Automated where possible (autoscaling replacement, leader re-election), manual where required. Time-to-recover is what matters; everything is broken eventually.

<img src="/diagrams/scalability/fault-tolerance-1.svg" alt="Active-active vs active-passive redundancy" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

## Redundancy patterns

**Active-active.** Multiple instances all handle traffic simultaneously. Failure of one is invisible; you just have less capacity. This is the default for stateless services behind a load balancer.

**Active-passive (warm standby).** Primary handles all traffic; secondary is warm and ready. On failure, traffic flips to secondary. Slower failover than active-active, used when the workload doesn't parallelize cleanly (e.g., some legacy databases).

**Active-passive (cold standby).** Secondary is off until needed. Cheaper but slower to take over. Used for disaster recovery, not normal failover.

**N+1 / N+2.** Capacity planning: provision enough that you can lose 1 (or 2) instances and still meet load. Mention this when sizing.

For databases and stateful tiers, redundancy is **replication**. See [Replication](/data/replication/).

## Failure isolation

Stopping a localized failure from becoming a global one is half the battle.

**Bulkheads.** Borrowed from ship design — partition resources so a flood in one compartment doesn't sink the ship. In software: separate thread pools, connection pools, or even servers per dependency. If the payment API stalls, only the threads dedicated to payment stall; everything else keeps moving.

<img src="/diagrams/scalability/fault-tolerance-2.svg" alt="Circuit breaker state machine: closed, open, half-open" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

**Circuit breakers.** When calls to a downstream service start failing, *stop calling it* for a period. Three states: closed (normal), open (failing fast for everyone), half-open (testing recovery with a trickle). Prevents the dying service from being completely buried in retries. Hystrix popularized the pattern; resilience4j and Polly are modern implementations.

**Timeouts.** Every cross-service call needs one. The classic outage is two services in a retry loop blocking forever — both eventually exhaust threads. **No default timeout = wrong timeout.** A sensible internal-call timeout is on the order of 1–5 seconds, with the budget enforced end-to-end (a request that's already burned 4 of its 5 seconds shouldn't make another 5-second downstream call).

**Cell-based architecture.** Take the bulkhead idea and apply it to the whole service: partition users into independent "cells," each with its own slice of databases, caches, and workers. A bad deploy or hot tenant in one cell affects only that cell's users. Slack, AWS, and Shopify all use variants of this.

## Retries (and why they're dangerous)

Retries are how you turn transient errors into successes. Done badly, they're how you turn a degraded service into a dead one.

Rules of thumb:

- **Only retry idempotent operations.** Retrying a non-idempotent write on a 500 may double-charge the user. Either make the operation idempotent (via idempotency keys) or don't retry.
- **Exponential backoff with jitter.** Without jitter, every retry from every client lines up at the same moment and stampedes the recovering service.
- **Bounded retries.** Never infinite. Three is usually enough.
- **Retry budget.** Track what fraction of calls are retries; if it exceeds a threshold (often 10%), stop retrying. Otherwise the retry traffic itself takes the service down.

In an interview, mention "exponential backoff with jitter and a retry budget" the first time you mention retries. It's the giveaway that you've seen real production.

## Failure detection

**Health checks.** Active probes from load balancers, orchestrators, or monitoring systems. Two flavors that matter:

- **Liveness probe.** Is the process up? Restart if not.
- **Readiness probe.** Is the process ready to take traffic? Remove from LB if not, but don't restart.

The difference matters in interviews. Most candidates conflate them.

**Heartbeats.** Components periodically tell a central coordinator "I'm alive." Missing heartbeats trigger replacement.

**Distributed consensus.** For "is this node the leader?" decisions, you need real consensus (Raft, Paxos). Don't roll your own; use etcd, ZooKeeper, or a managed service.

## Graceful degradation

When something *is* broken, the user-facing question is: do you fail completely, or do you fall back?

- **Read from a cache when the DB is down.** Stale data is usually better than no data.
- **Show a generic feed when the personalized one fails.** Most users won't notice.
- **Drop low-priority traffic first.** Analytics events can be sampled; user-facing reads can't.
- **Read-only mode.** A degraded but functional system beats a 503.

In interviews, naming a graceful degradation path for a critical dependency is high-signal.

## The math of nines

You will be asked about SLOs and uptime. The basics:

| Availability | Downtime per year | Downtime per month |
|--------------|-------------------|--------------------|
| 99% | 3.65 days | 7.2 hours |
| 99.9% ("three nines") | 8.76 hours | 43.2 minutes |
| 99.95% | 4.38 hours | 21.6 minutes |
| 99.99% ("four nines") | 52.6 minutes | 4.32 minutes |
| 99.999% ("five nines") | 5.26 minutes | 25.9 seconds |

Two consequences worth knowing:

- **Series multiplies.** If a request depends on four services each at 99.9%, the end-to-end is 0.999⁴ ≈ 99.6%. Stacking dependencies costs availability.
- **Parallel adds.** Two redundant copies each at 99.9% give you 1 − (0.001)² = 99.9999%. Redundancy is how you climb the nines.

You can ground "we need 99.95%" claims in real arithmetic instead of vibes. That always sounds senior.

## What to say in an interview

A clean fault-tolerance paragraph:

> *"Every stateless service runs in active-active across three availability zones, with N+1 capacity. Cross-service calls have a 2-second timeout, exponential-backoff retries capped at 3, and a circuit breaker that opens at 50% error rate. The database is a primary with semi-sync replicas in two other AZs; failover is automated via Patroni with ~30-second RTO. If Redis is down, reads degrade to direct database hits at higher latency rather than failing. Our SLO target is 99.95%, which the math gives us at this redundancy level."_*

Eight concrete decisions, all named. That's the bar for a fault-tolerance deep dive.

## Common pitfalls

**Single point of failure hiding in a "highly available" diagram.** That one Redis. That one DNS provider. That one bastion host. Walk the diagram and ask "what dies if this single box dies?" for each box.

**Retries without backoff.** Free DoS on your own service.

**No timeout, or timeouts longer than the budget.** A 30-second downstream timeout inside a 10-second user-facing budget is dead on arrival.

**Untested failover.** A failover path you've never exercised is a failover path that doesn't work. Chaos engineering exists for this reason.

**Treating "redundancy" as a checkbox.** Two replicas in the same rack share a switch; two AZs in the same region share a control plane. Redundancy is only as good as the failure domains it crosses.

If you can name your failure modes and the mitigation for each, you've done the work this section is testing.
