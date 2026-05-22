---
title: Load Balancers
description: Layer 4 vs layer 7, algorithms, health checks, sticky sessions, and how load balancers fit into a real architecture — the parts that come up in every system design interview.
---

A load balancer (LB) sits between clients and a pool of servers, accepting incoming connections and distributing them across the pool. It is the single most universally useful box in system design: nearly every diagram has one, and interviewers expect you to know it well enough to defend the algorithm, the layer, and the failure modes.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 260" role="img" aria-label="Layer 4 vs Layer 7 load balancing side by side" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Layer 4 vs Layer 7 load balancing</text>
  <g transform="translate(20,45)">
    <rect width="290" height="200" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="145" y="28" text-anchor="middle" fill="currentColor" font-weight="700">Layer 4 (TCP/UDP)</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="60" width="80" height="30" rx="6"/>
      <rect x="120" y="60" width="50" height="30" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="190" y="40" width="80" height="22" rx="6"/>
      <rect x="190" y="68" width="80" height="22" rx="6"/>
      <rect x="190" y="96" width="80" height="22" rx="6"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="60" y="79">Client</text>
      <text x="145" y="79" font-weight="600">L4 LB</text>
      <text x="230" y="55">backend</text>
      <text x="230" y="83">backend</text>
      <text x="230" y="111">backend</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M100 75 H120"/>
      <path d="M170 75 L190 51"/>
      <path d="M170 75 H190"/>
      <path d="M170 75 L190 107"/>
    </g>
    <g fill="currentColor" font-size="11">
      <text x="20" y="145" font-weight="600">Sees:</text>
      <text x="60" y="145">IPs &amp; ports only</text>
      <text x="20" y="165" font-weight="600">Good for:</text>
      <text x="80" y="165">raw throughput, gRPC,</text>
      <text x="20" y="181">databases, custom protocols</text>
    </g>
  </g>
  <g transform="translate(330,45)">
    <rect width="290" height="200" rx="10" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2"/>
    <text x="145" y="28" text-anchor="middle" fill="currentColor" font-weight="700">Layer 7 (HTTP)</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="20" y="60" width="80" height="30" rx="6"/>
      <rect x="120" y="60" width="50" height="30" rx="6" fill="var(--sl-color-accent,#3b82f6)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="190" y="40" width="80" height="22" rx="6"/>
      <rect x="190" y="68" width="80" height="22" rx="6"/>
      <rect x="190" y="96" width="80" height="22" rx="6"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="60" y="79">Client</text>
      <text x="145" y="79" font-weight="600" fill="var(--sl-color-white,#fff)">L7 LB</text>
      <text x="230" y="55">/users</text>
      <text x="230" y="83">/orders</text>
      <text x="230" y="111">/static</text>
    </g>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <path d="M100 75 H120"/>
      <path d="M170 75 L190 51"/>
      <path d="M170 75 H190"/>
      <path d="M170 75 L190 107"/>
    </g>
    <g fill="currentColor" font-size="11">
      <text x="20" y="145" font-weight="600">Sees:</text>
      <text x="60" y="145">URL, headers, cookies</text>
      <text x="20" y="165" font-weight="600">Good for:</text>
      <text x="80" y="165">path routing, TLS, A/B,</text>
      <text x="20" y="181">retries, compression</text>
    </g>
  </g>
</svg>

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

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 220" role="img" aria-label="Round robin vs least connections, with one slow backend" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">Why "least connections" beats round robin with uneven backends</text>
  <g transform="translate(20,50)">
    <text x="145" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Round robin</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="30" y="15" width="80" height="35" rx="6"/>
      <rect x="120" y="15" width="80" height="35" rx="6"/>
      <rect x="210" y="15" width="80" height="35" rx="6"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="70" y="37">A: 5 reqs</text>
      <text x="160" y="37">B: 5 reqs</text>
      <text x="250" y="37">C: 5 reqs</text>
    </g>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="30" y="65" width="80" height="60" rx="6"/>
      <rect x="120" y="65" width="80" height="60" rx="6"/>
      <rect x="210" y="65" width="80" height="100" rx="6" fill="var(--sl-color-accent,#3b82f6)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="70" y="100">2 in-flight</text>
      <text x="160" y="100">2 in-flight</text>
      <text x="250" y="115" fill="var(--sl-color-white,#fff)">C slow:</text>
      <text x="250" y="130" fill="var(--sl-color-white,#fff)">5 in-flight</text>
    </g>
    <text x="145" y="190" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">C backs up; tail latency spikes.</text>
  </g>
  <g transform="translate(340,50)">
    <text x="145" y="0" text-anchor="middle" fill="currentColor" font-weight="600">Least connections</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="30" y="15" width="80" height="35" rx="6"/>
      <rect x="120" y="15" width="80" height="35" rx="6"/>
      <rect x="210" y="15" width="80" height="35" rx="6"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="70" y="37">A: 7 reqs</text>
      <text x="160" y="37">B: 7 reqs</text>
      <text x="250" y="37">C: 1 req</text>
    </g>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="30" y="65" width="80" height="80" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="120" y="65" width="80" height="80" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
      <rect x="210" y="65" width="80" height="80" rx="6" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="11">
      <text x="70" y="110">3 in-flight</text>
      <text x="160" y="110">3 in-flight</text>
      <text x="250" y="110">3 in-flight</text>
    </g>
    <text x="145" y="190" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.8">LB routes new reqs to the freer backend.</text>
  </g>
</svg>

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

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 320" role="img" aria-label="Typical layered load balancer architecture from client to app servers" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="270" y="10" width="100" height="35" rx="8"/>
    <rect x="270" y="65" width="100" height="35" rx="8"/>
    <rect x="240" y="125" width="160" height="40" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="240" y="190" width="160" height="40" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="100" y="260" width="120" height="40" rx="8"/>
    <rect x="260" y="260" width="120" height="40" rx="8"/>
    <rect x="420" y="260" width="120" height="40" rx="8"/>
  </g>
  <g fill="currentColor" text-anchor="middle">
    <text x="320" y="32">Client</text>
    <text x="320" y="87">DNS / Anycast</text>
    <text x="320" y="150" font-weight="600">Edge L4 LB</text>
    <text x="320" y="215" font-weight="600">L7 LB / Gateway</text>
    <text x="160" y="285">App svc A</text>
    <text x="320" y="285">App svc B</text>
    <text x="480" y="285">App svc C</text>
  </g>
  <g font-size="11" fill="currentColor" opacity="0.8">
    <text x="410" y="148">TLS, DDoS</text>
    <text x="410" y="213">routing, auth, RL</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none">
    <path d="M320 45 V65"/>
    <path d="M320 100 V125"/>
    <path d="M320 165 V190"/>
    <path d="M320 230 V245 H160 V260"/>
    <path d="M320 245 V260"/>
    <path d="M320 245 H480 V260"/>
  </g>
  <g fill="currentColor">
    <polygon points="316,67 320,61 324,67"/>
    <polygon points="316,127 320,121 324,127"/>
    <polygon points="316,192 320,186 324,192"/>
  </g>
</svg>


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
