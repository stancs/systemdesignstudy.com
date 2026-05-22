---
title: Load Balancers
description: Layer 4 vs layer 7, algorithms, health checks, sticky sessions, and how load balancers fit into a real architecture — the parts that come up in every system design interview.
---

A load balancer (LB) sits between clients and a pool of servers, accepting incoming connections and distributing them across the pool. It is the single most universally useful box in system design: nearly every diagram has one, and interviewers expect you to know it well enough to defend the algorithm, the layer, and the failure modes.

<img src="/diagrams/networking/load-balancers-1.svg" alt="Layer 4 vs Layer 7 load balancing side by side" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

## Layer 4 vs layer 7

The first decision is which OSI layer to balance at.

**Layer 4 (transport / TCP, UDP).** The LB sees only IPs and ports. It picks a backend, opens a connection, and shovels bytes. It does not parse the payload. Layer-4 balancers are fast, cheap, and protocol-agnostic — they can balance anything (databases, gRPC, custom binary protocols). Examples: AWS NLB, HAProxy in TCP mode, IPVS.

**Layer 7 (application / HTTP).** The LB parses the request. It can route by URL path, header, cookie, method, or body. It can terminate TLS, do compression, retry idempotent requests, and inject headers. Examples: AWS ALB, Nginx, Envoy, Cloudflare.

A useful rule of thumb:

- Reach for **L4** when you need raw throughput or are balancing non-HTTP traffic.
- Reach for **L7** when you want path-based routing, A/B testing, header-based feature flags, or any kind of request-aware logic.

Most real systems use both: an L4 LB at the edge for raw connection acceptance, an L7 LB inside the cluster for application routing.

## Algorithms

You will probably be asked to pick one and defend it.

- **Round robin.** Send each new connection to the next backend in the ring. Simple, good when backends are interchangeable and requests are similar.
- **Least connections.** Send the next request to the backend with the fewest open connections. Better when requests have variable cost or backends have different capacity.
- **Least response time.** Combines least-connections with latency. Most effective when backends behave heterogeneously.
- **Weighted variants.** Assign each backend a weight (round robin, least connections). Use when running mixed instance sizes or canarying a new version.
- **IP hash / consistent hash.** Send all requests from the same client (or with the same key) to the same backend. Useful for in-memory caching on the backend or sticky session needs. See [Consistent Hashing](/scalability/consistent-hashing/).
- **Random with two choices (P2C).** Pick two backends at random and send to whichever has fewer connections. Surprisingly close to least-connections at a fraction of the coordination cost.

In an interview, *least connections* is the safest default for application traffic; *consistent hash* is what you mention when you need cache locality or session affinity.

<img src="/diagrams/networking/load-balancers-2.svg" alt="Round robin vs least connections, with one slow backend" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

## Health checks

A load balancer is only useful if it can detect and remove unhealthy backends. Two flavors:

- **Active health checks.** The LB periodically pings each backend (e.g., `GET /healthz`). Simple, works on quiet backends, but adds load.
- **Passive health checks.** The LB watches real traffic — too many timeouts or 5xxs and the backend gets marked unhealthy. No extra load, but slower to react on low-traffic services.

Production setups usually combine the two. The key parameters to mention are:

- **Interval** — how often to check.
- **Threshold** — how many consecutive failures before marking unhealthy (and successes before marking healthy again).
- **Timeout** — how long to wait per check.

Healthy/unhealthy state should be **slow to flip both ways** so flapping backends do not whipsaw traffic.

## Stickiness (session affinity)

Sometimes you want all requests from a given user to land on the same backend — for example, when the backend keeps in-memory session state or a per-user cache. Options:

- **Cookie-based.** The LB sets a cookie on the first response identifying the backend; subsequent requests are routed accordingly.
- **IP-based.** Hash the client IP. Cheap, but breaks behind NAT and corporate proxies.
- **Consistent hash on a request key.** Route by user ID, session ID, etc.

Sticky sessions are a tax. They couple users to specific machines, hurt failover, and make rolling deploys harder. In an interview, prefer **stateless backends with externalized session state** (Redis, signed JWTs). Only reach for stickiness when the cost of externalizing state is genuinely higher than the cost of pinning.

## Where the load balancer sits

A typical, defensible diagram looks like this:

<img src="/diagrams/networking/load-balancers-3.svg" alt="Typical layered load balancer architecture from client to app servers" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>


For internal service-to-service traffic, you'll often have a second tier of L7 LBs (or a service mesh sidecar) inside the cluster. The same principles apply.

## Common interview deep dives

**How do you make the load balancer itself highly available?**

A single LB is a single point of failure. Standard answers: run an **active-active pair** with floating IPs (keepalived/VRRP), use a managed LB whose control plane handles failover (ALB, GCLB), or run **Anycast LB nodes** so the network reroutes traffic when one node disappears.

**What happens during a deploy?**

The LB should drain connections from the backend being replaced — stop sending new requests but let existing ones finish for some grace period (15–60s typical). Combined with readiness probes, this avoids spilling errors during rolling deploys.

**How do you handle a thundering herd of new connections?**

L4 LBs can handle millions of connections; L7 LBs are usually the constraint. You can prewarm capacity, lean on connection-reuse (HTTP/2 multiplexing), and use queueing or rate limiting at the LB level to shed load gracefully rather than collapsing.

**Can the LB cause hot spots?**

Yes — especially with consistent-hash routing and skewed keys. The classic fix is to **add virtual nodes** so each backend handles many hash positions, smoothing the distribution.

## What to say in an interview

For most prompts, two sentences is enough:

> *"Clients hit an L4 edge LB that terminates TLS and forwards to a per-region L7 LB. The L7 LB routes by path to the right service and uses least-connections with passive health checks; sticky sessions are off because session state lives in Redis."*

The instant you reach for any non-default — consistent hashing, sticky sessions, weighted backends — pair it with the reason. The reason is what gets graded.
