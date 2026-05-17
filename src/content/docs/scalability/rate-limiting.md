---
title: Rate Limiting
description: Why you rate limit, the four algorithms worth knowing (token bucket, leaky bucket, fixed window, sliding window), and where in the stack to enforce them.
---

Rate limiting is the practice of capping how often a client can hit your system. It protects you from abuse, accidental bugs, runaway scripts, and from one tenant burning the resources of others. Almost every production API has it, and "design a rate limiter" is a frequent system design interview prompt in its own right.

This page covers the algorithms, the architecture, and the gotchas.

## Why you rate limit

In rough priority order:

- **Abuse and DoS protection.** Stop attackers and aggressive bots from saturating your infrastructure.
- **Fairness across tenants.** Stop one big customer from starving everyone else.
- **Cost control.** Stop a misbehaving caller from running up bills (compute, third-party APIs, egress bandwidth).
- **Quota enforcement.** Implement the rate limits your pricing tiers actually advertise.
- **Backpressure.** Slow the front door when a downstream system is in trouble.

If you don't have a rate limiter in your design and the prompt has a public API, mention it.

## The four algorithms

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 240" role="img" aria-label="Token bucket: refills at rate r, each request takes a token" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Token bucket</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="220" y="60" width="160" height="140" rx="10"/>
    <line x1="220" y1="90" x2="380" y2="90" stroke-dasharray="3 3"/>
  </g>
  <g fill="var(--sl-color-accent,#3b82f6)">
    <circle cx="250" cy="120" r="10"/>
    <circle cx="280" cy="135" r="10"/>
    <circle cx="320" cy="125" r="10"/>
    <circle cx="350" cy="140" r="10"/>
    <circle cx="265" cy="160" r="10"/>
    <circle cx="305" cy="165" r="10"/>
    <circle cx="340" cy="175" r="10"/>
  </g>
  <text x="300" y="80" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">capacity N (burst limit)</text>
  <g fill="currentColor" text-anchor="middle">
    <text x="300" y="225" font-weight="600">Bucket</text>
  </g>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="60" y="105" width="100" height="40" rx="8"/>
    <rect x="440" y="105" width="140" height="40" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
  </g>
  <g fill="currentColor" text-anchor="middle" font-size="12">
    <text x="110" y="125" font-weight="600">Refill</text>
    <text x="110" y="141" font-size="11">r tokens/sec</text>
    <text x="510" y="125" font-weight="600">Request</text>
    <text x="510" y="141" font-size="11">consumes 1 token</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none">
    <path d="M160 125 H220"/>
    <path d="M380 125 H440"/>
  </g>
  <g fill="currentColor">
    <polygon points="216,123 222,125 216,127"/>
    <polygon points="436,123 442,125 436,127"/>
  </g>
</svg>

### Token bucket

A bucket holds up to N tokens. Each request consumes one token. Tokens refill at a fixed rate (r tokens/sec). If the bucket is empty, the request is rejected (or queued, or shed).

**Properties**

- Allows **bursts** up to N requests, then settles into r requests/sec.
- Simple, widely used, easy to explain.
- Per-key state is just two numbers: current tokens + last refill timestamp.

**Where you see it.** AWS, Stripe, most cloud APIs.

### Leaky bucket

A queue with bounded capacity drains at a fixed rate. Requests enter the queue; if the queue is full, they're rejected. Requests leave the queue at a constant rate.

**Properties**

- Smooths bursts entirely — output rate is strictly constant regardless of input shape.
- Higher latency under load (requests sit in the queue).
- Best when you want a downstream service to receive a perfectly steady stream.

**Where you see it.** Network gear, anything sensitive to bursty traffic patterns.

The difference between token bucket and leaky bucket is whether the limit is on **arrivals** (token) or **departures** (leaky). Most user-facing limits use token bucket because the burst tolerance is desirable.

### Fixed window counter

Count requests in fixed time buckets — say, requests in this minute. Cap at N. Reset the counter when the bucket rolls.

**Properties**

- Trivial to implement (`INCR` in Redis, with TTL of one window).
- **Edge problem.** A client can send N requests in the last second of one window and N more in the first second of the next, getting 2N in two seconds. For a 100/min limit, that's 200/2 sec — a 12x burst over what you advertised.

Fine for rough quotas, less appropriate as a precise rate enforcer.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 260" role="img" aria-label="Fixed window edge problem vs sliding window" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Fixed window has an edge problem; sliding window doesn't</text>
  <g transform="translate(0,40)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Fixed window (100/min)</text>
    <g stroke="currentColor" stroke-width="2" fill="none">
      <line x1="20" y1="80" x2="300" y2="80"/>
      <line x1="160" y1="70" x2="160" y2="90"/>
      <line x1="20" y1="70" x2="20" y2="90"/>
      <line x1="300" y1="70" x2="300" y2="90"/>
    </g>
    <g fill="var(--sl-color-accent,#3b82f6)" opacity="0.7">
      <rect x="140" y="50" width="20" height="30"/>
      <rect x="160" y="50" width="20" height="30"/>
    </g>
    <g fill="currentColor" font-size="11" text-anchor="middle">
      <text x="90" y="105">window 1</text>
      <text x="230" y="105">window 2</text>
      <text x="160" y="40" fill="var(--sl-color-accent,#3b82f6)" font-weight="600">200 reqs in ~2 sec</text>
    </g>
    <text x="160" y="160" text-anchor="middle" fill="var(--sl-color-accent,#3b82f6)" font-size="11">Client can stack 100+100 across the boundary.</text>
  </g>
  <g transform="translate(320,40)">
    <text x="160" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Sliding window (100/min)</text>
    <g stroke="currentColor" stroke-width="2" fill="none">
      <line x1="20" y1="80" x2="300" y2="80"/>
      <rect x="100" y="40" width="120" height="50" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2"/>
    </g>
    <g fill="currentColor" font-size="11" text-anchor="middle">
      <text x="160" y="65" font-weight="600">last 60 sec</text>
      <text x="160" y="105">window slides with time</text>
    </g>
    <text x="160" y="160" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">No edge effect — always counts the last 60 s.</text>
  </g>
</svg>

### Sliding window log / sliding window counter

Keep a log of timestamps of recent requests; count the ones within the last window. Reject if count ≥ N.

**Properties**

- Solves the edge problem of fixed windows — you can't double-burst at the boundary.
- More expensive to maintain than fixed windows. Pure log is O(requests) memory per key; sliding window counter is a cheap approximation using two adjacent fixed windows weighted by their overlap.

The sliding window counter approximation is what most production rate limiters actually use. It's nearly as accurate as a full log at a fraction of the cost.

## A decision cheatsheet

| If you want… | Use… |
|--------------|------|
| Burst tolerance with a steady-state cap | Token bucket |
| Perfectly smooth output rate | Leaky bucket |
| Cheap, approximate quota | Fixed window counter |
| Fair limiting near window boundaries | Sliding window counter |

In an interview, **token bucket** is the safest default. It's the easiest to explain, matches user expectations of "bursts okay, sustained abuse not okay," and is what most public APIs implement.

## Where in the stack to enforce

You can rate limit at every layer; pick where it does the most good.

- **CDN / edge.** Cheapest place to drop traffic — it never enters your network. Best for crude protections (per-IP DDoS-class limits).
- **API gateway.** Per-user / per-API-key / per-endpoint limits. The natural home for application rate limits. See [API Gateway](/networking/api-gateway/).
- **Application server.** Last line of defense; useful for limits that depend on application state (e.g., "max 5 concurrent file uploads per user").
- **Database connection pool.** A different kind of rate limit (concurrency) — caps connections rather than requests.

A common shape: crude IP-based limits at the edge, per-user limits at the gateway, per-feature limits in the application.

## Distributed rate limiting

A single rate limiter on one box is easy. Once you have N gateway instances, you need them to share state — otherwise each instance enforces 1/N of the real limit. Three common approaches:

**Central store (Redis).** All instances increment counters in a shared Redis. Simple, accurate, and the Redis becomes a hot spot under load. Mitigations: shard by user_id, use Redis Cluster, batch increments.

**Local + reconciliation.** Each instance enforces its own slice of the limit locally; periodically reconcile with a central store. Less accurate (transient over-limit possible), much higher throughput.

**Probabilistic structures.** Use approximate counters (count-min sketch) to scale to enormous keyspaces with bounded memory.

For most systems, "Redis with INCR + EXPIRE" is the right answer until proven otherwise. Mention it.

## Identifiers: what you key on

What you key the limit on determines what attack it stops:

- **IP address.** Stops basic bots; useless against distributed attacks; harms users behind NAT.
- **User ID.** Stops a logged-in user from hammering you. Requires authentication first.
- **API key / OAuth client.** The natural unit for partner APIs.
- **Endpoint + identifier.** Different limits per endpoint. Cheap reads might be 1000/min, expensive writes 10/min.

Most production systems layer multiple keys: a low per-IP limit at the edge plus per-user and per-endpoint limits at the gateway.

## What to say when you reject

Two parts of the response matter:

- **Status code.** Use **HTTP 429 Too Many Requests**.
- **`Retry-After` header.** Tells clients how long to wait — in seconds (`Retry-After: 30`) or as an HTTP date.

Optionally include rate-limit context in headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) so well-behaved clients can self-regulate. GitHub and Stripe both do this and it's worth copying.

## Soft vs hard limits, and graceful degradation

A blanket "reject above limit" is the default, but two refinements come up:

- **Soft limits with backoff.** Above limit, slow down rather than reject. Add latency, then drop only the truly persistent offenders.
- **Tiered priorities.** Premium customers get a larger bucket; free tier gets the smaller one. Same algorithm, per-tier parameters.

If the prompt is a SaaS API with paid tiers, mention this.

## Common pitfalls

**Limiting the wrong thing.** Rate-limiting by IP for an API with corporate customers locks out everyone behind one office NAT. Pick keys appropriate to your audience.

**Forgetting WebSockets and long polls.** A connected user holds an open connection; you can't apply a per-request limit. Limit *messages* on the WebSocket, not connections.

**Burst tolerance for the wrong endpoint.** A token bucket is wrong for an endpoint where each call is genuinely expensive (e.g., AI inference). Use a smaller bucket or leaky-bucket smoothing.

**No DLQ for rejected work.** If rate-limited requests represent real user work, dropping them silently is bad UX. Either return 429 so the client can retry, or enqueue them for later processing — but don't blackhole.

## What to say in an interview

A clean rate-limit paragraph that lands well:

> *"At the gateway we enforce a token-bucket rate limit per user_id — 1000 requests/minute steady-state with a 200-request burst. State lives in Redis using `INCR` with a TTL of the window. Limits are tiered by plan, so paid users get a larger bucket. When clients exceed, we return 429 with a `Retry-After` header; persistent offenders get throttled at the edge by IP instead, to keep them out of our network entirely. We also alert when any single user_id sustains 80% of their limit for an hour — that's usually a bug, not malice."*

Five concrete decisions in a paragraph. That is the bar.
