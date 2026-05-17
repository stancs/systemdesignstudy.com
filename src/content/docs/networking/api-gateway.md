---
title: API Gateway
description: What an API gateway does, when to introduce one, how it differs from a load balancer, and the cross-cutting concerns it earns its keep on.
---

An API gateway is the single front door for client traffic into a backend system. It sits in front of one or more services and handles the *cross-cutting concerns* that would otherwise be duplicated in every service: authentication, rate limiting, request routing, protocol translation, observability, and response shaping.

If a load balancer is plumbing, an API gateway is a control plane.

## Load balancer vs gateway

Candidates often confuse the two. The clearest distinction:

- A **load balancer** decides *which backend instance* gets a request.
- An **API gateway** decides *what to do with the request* before any backend sees it.

In practice they overlap — modern L7 LBs (Envoy, Nginx, ALB) do gateway-ish things, and modern gateways do load balancing. But the conceptual split matters: the LB is about distributing connections; the gateway is about enforcing policy.

You almost always have both in a real architecture. The LB terminates the connection and picks an instance of the gateway; the gateway then does the policy work and forwards to the right downstream service.

## What an API gateway actually does

Six things, in rough order of how often they come up in interviews:

**1. Authentication and authorization.** The gateway validates tokens (JWT, OAuth, API keys), enriches the request with user identity, and rejects everything else. Doing this once at the edge is dramatically cheaper than re-validating in every service.

**2. Rate limiting and quotas.** Per-user, per-IP, per-API-key, per-endpoint. See [Rate Limiting](/scalability/rate-limiting/). Centralizing this at the gateway lets you enforce limits before the request consumes any downstream resources.

**3. Routing and request transformation.** Map a public API surface to one or more internal services. `GET /v1/users/123` might fan out to a user service, a profile service, and a permissions service, and return a single composed response. This is often called the **Backend-for-Frontend (BFF)** pattern.

**4. Protocol translation.** The gateway speaks HTTP/JSON to the world and gRPC, WebSockets, or queues to the inside. Mobile clients particularly benefit because gateways can compose what would otherwise be many round trips into one.

**5. Observability.** Every request flows through one place. Log structured request metadata, propagate trace IDs, emit metrics by route, capture latency. The gateway becomes the obvious place to attach access logs and dashboards.

**6. Caching, compression, TLS.** Many gateways do response caching for read-heavy endpoints, gzip/brotli compression, and TLS termination so the inside of the cluster can run in plaintext.

## When you do — and don't — need one

Reach for an API gateway when:

- You have **more than a handful of services** and don't want to re-implement auth, rate limiting, and logging in each one.
- You support **multiple client types** (web, iOS, Android, partners) and want a thin compatibility layer between them and your services.
- You expose a **public API** with quotas, billing, or developer keys.

Skip the gateway (or use a thin L7 LB instead) when:

- The system is a single monolith. The "gateway" is just middleware in the app.
- You only have two or three services and the operational overhead of another box isn't worth it.

Interviewers reward this kind of pragmatism. *"A gateway would be overkill at this scale; we'll keep auth in a shared middleware library inside the monolith and revisit when we split out the first internal service"* is a senior-sounding answer.

## Popular implementations

Worth name-dropping if relevant:

- **Managed**: AWS API Gateway, Google Cloud API Gateway, Azure API Management, Cloudflare Workers, Kong Cloud.
- **Self-hosted, open-source**: Kong, Tyk, KrakenD, Envoy (configured as a gateway), Apollo Router (for GraphQL).
- **Service-mesh-as-gateway**: Istio's ingress gateway, Linkerd's gateway integration.

For most interview answers, "API Gateway" without a brand is fine. Naming a specific one only helps if you can defend the choice ("Kong because we want a self-hosted, plugin-extensible gateway and don't want to pay AWS API Gateway per-request").

## The Backend-for-Frontend pattern

A common gateway pattern in interviews:

```
[iOS app]    -> [iOS BFF]    \
[Web app]    -> [Web BFF]     ->  [User svc] [Order svc] [Catalog svc]
[Partner API]-> [Public BFF] /
```

Each BFF is shaped for the needs of its client. The iOS BFF may return denormalized responses tailored for mobile screens; the partner BFF may speak a versioned, more conservative shape. Each BFF reuses the same underlying services.

The trade-off is duplication: every client now has its own BFF to maintain. The win is that **no single gateway becomes a god service** trying to please everyone.

## Common pitfalls

**Making the gateway a microservice in disguise.** If your gateway starts doing meaningful business logic, you've reinvented the monolith — but worse, because every change ships through a piece of infrastructure rather than an application.

**Single point of failure.** Run multiple gateway instances behind the LB. Health-check them. Plan capacity.

**Hot path latency.** The gateway is in every request's critical path. Every plugin you add costs latency for every request. Profile, prune, and tune aggressively. A 5ms-per-plugin overhead with ten plugins is 50ms baseline before any work.

**Auth-in-gateway, auth-in-service mismatch.** If the gateway strips auth and the service trusts a header, anyone who can reach the service directly bypasses your security. Lock down internal network paths, or have services re-validate critical claims.

## What to say in an interview

A solid one-liner when introducing the gateway:

> *"In front of the services we run an API gateway. It terminates TLS, validates the user's JWT, enforces per-user rate limits, and routes by path to the right downstream service. For mobile clients we run a thin BFF on the gateway that composes the user, feed, and notification responses into a single payload so the client doesn't pay three round trips."*

The instant the interviewer asks "why a gateway?" you have your answer: it owns the cross-cutting concerns so the services don't have to.
