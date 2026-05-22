---
title: DNS & The Client-Server Model
description: How a request finds your servers — DNS resolution, the role of TTLs, Anycast, and the parts of the client-server model that matter in interviews.
---

Before any of your servers see a packet, a client has to figure out *where to send it*. That work happens at the very edge — and it shows up in system design interviews more often than candidates expect, especially when the question turns to global scale, multi-region failover, or CDN strategy.

## The client-server model in one paragraph

A client (browser, mobile app, another service) wants a resource. It opens a connection to a server identified by an IP address, sends a request, and gets a response. Between the client and the server there may be a load balancer, an API gateway, a CDN, a service mesh, and several layers of caching. Every box in your interview diagram exists either to *resolve, route, or serve* requests faster, cheaper, or more reliably than a direct connection would.

The key idea is that the system is **fundamentally pull-based**: the client initiates and the server responds. Notifications, websockets, and server-sent events bend this rule but don't break it — the connection is still established by the client.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 680 220" role="img" aria-label="DNS resolution chain from browser to authoritative nameserver" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:12px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <defs>
    <marker id="dns-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto">
      <path d="M0,1 L9,5 L0,9 z" fill="currentColor"/>
    </marker>
  </defs>
  <text x="340" y="22" text-anchor="middle" fill="currentColor" font-weight="600">A DNS lookup, end to end</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="20" y="80" width="100" height="50" rx="8" fill="var(--sl-color-accent-low)" stroke="var(--sl-color-accent)" stroke-width="2"/>
    <rect x="150" y="80" width="100" height="50" rx="8"/>
    <rect x="280" y="80" width="100" height="50" rx="8"/>
    <rect x="410" y="80" width="100" height="50" rx="8"/>
    <rect x="540" y="80" width="120" height="50" rx="8" fill="var(--sl-color-accent-low)" stroke="var(--sl-color-accent)" stroke-width="2"/>
  </g>
  <g fill="currentColor" text-anchor="middle">
    <text x="70" y="100" font-weight="600">Client</text>
    <text x="70" y="118" font-size="11">browser + OS</text>
    <text x="200" y="100" font-weight="600">Recursive</text>
    <text x="200" y="118" font-size="11">resolver</text>
    <text x="330" y="100" font-weight="600">Root</text>
    <text x="330" y="118" font-size="11">NS</text>
    <text x="460" y="100" font-weight="600">TLD</text>
    <text x="460" y="118" font-size="11">NS (.com)</text>
    <text x="600" y="100" font-weight="600">Authoritative</text>
    <text x="600" y="118" font-size="11">example.com</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none" marker-end="url(#dns-arrow)">
    <path d="M122 105 H148"/>
    <path d="M252 105 H278"/>
    <path d="M382 105 H408"/>
    <path d="M512 105 H538"/>
  </g>
  <g font-size="11" fill="currentColor" opacity="0.8" text-anchor="middle">
    <text x="135" y="155">1. cache miss</text>
    <text x="265" y="155">2. ask root</text>
    <text x="395" y="155">3. ask .com</text>
    <text x="525" y="155">4. ask zone</text>
  </g>
  <text x="340" y="200" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.7">Each layer caches by TTL — most lookups stop at the recursive resolver.</text>
</svg>

## DNS, end to end

DNS (Domain Name System) is the directory that turns a human name like `api.example.com` into a routable IP address. A typical lookup goes:

1. **Browser cache** — most browsers cache resolutions for a few minutes.
2. **OS stub resolver** — the operating system has its own cache.
3. **Recursive resolver** (e.g., your ISP's, or 1.1.1.1, or 8.8.8.8). If it has the answer cached, it returns it.
4. **Root nameservers** — direct the resolver to the right TLD nameserver (`.com`, `.org`, …).
5. **TLD nameservers** — direct the resolver to the authoritative nameserver for `example.com`.
6. **Authoritative nameserver** — returns the actual record (A, AAAA, CNAME, etc.).

In an interview you do not need to recite this whole chain. You do need to know:

- **Resolution is cached at every layer.** That cache is governed by the **TTL** (time-to-live) on each record.
- **Low TTLs trade DNS load for agility.** A 30-second TTL means failover happens within ~30s but you generate 60x more DNS queries than a 30-minute TTL.
- **TTLs are not honored uniformly.** Some resolvers cap them; some browsers ignore them entirely. Plan as if some clients keep stale records for hours.

## The records you'll actually mention

- **A** — name to IPv4 address.
- **AAAA** — name to IPv6 address.
- **CNAME** — name to another name (alias). Cannot coexist with other records at the apex of a zone.
- **MX** — mail exchange records.
- **TXT** — arbitrary text, used for SPF/DKIM, ownership verification.
- **NS** — delegates a subdomain to another set of nameservers.

For modern, performant setups you'll also encounter:

- **ALIAS / ANAME** — provider-specific "CNAME-at-apex" records.
- **GeoDNS** — returns different answers based on the resolver's region.
- **Weighted records** — return different answers in proportion, for blue/green or canary.

## Anycast: one IP, many locations

A modern global service almost certainly fronts its edge with **Anycast**. The same IP address is advertised from multiple datacenters; routers send each packet to the *closest* advertiser by BGP. The client doesn't know there are dozens of edge nodes — it sees one IP.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 288" role="img" aria-label="Anycast: four users each routed to their nearest POP, all advertising the same IP" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <defs>
    <marker id="any-ah" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="11" markerHeight="11" markerUnits="userSpaceOnUse" orient="auto">
      <path d="M0,1 L9,5 L0,9 z" fill="currentColor"/>
    </marker>
  </defs>
  <text x="320" y="24" text-anchor="middle" fill="currentColor" font-weight="600">Anycast: one IP, many points of presence</text>

  <g fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2">
    <rect x="20" y="58" width="134" height="46" rx="10"/>
    <rect x="174" y="58" width="134" height="46" rx="10"/>
    <rect x="332" y="58" width="134" height="46" rx="10"/>
    <rect x="486" y="58" width="134" height="46" rx="10"/>
  </g>
  <g fill="currentColor" text-anchor="middle" font-size="12.5">
    <text x="87" y="86">User in SF</text>
    <text x="241" y="86">User in London</text>
    <text x="399" y="86">User in Tokyo</text>
    <text x="553" y="86">User in São Paulo</text>
  </g>

  <g stroke="currentColor" stroke-width="2" fill="none">
    <path d="M87 104 V160" marker-end="url(#any-ah)"/>
    <path d="M241 104 V160" marker-end="url(#any-ah)"/>
    <path d="M399 104 V160" marker-end="url(#any-ah)"/>
    <path d="M553 104 V160" marker-end="url(#any-ah)"/>
  </g>

  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="20" y="164" width="134" height="62" rx="10"/>
    <rect x="174" y="164" width="134" height="62" rx="10"/>
    <rect x="332" y="164" width="134" height="62" rx="10"/>
    <rect x="486" y="164" width="134" height="62" rx="10"/>
  </g>
  <g text-anchor="middle">
    <g fill="currentColor" font-weight="600" font-size="12.5">
      <text x="87" y="190">POP · US-West</text>
      <text x="241" y="190">POP · Europe</text>
      <text x="399" y="190">POP · Asia</text>
      <text x="553" y="190">POP · S. America</text>
    </g>
    <g fill="var(--sl-color-accent,#3b82f6)" font-weight="700" font-size="13" font-family="ui-monospace,SFMono-Regular,Menlo,monospace">
      <text x="87" y="212">203.0.113.7</text>
      <text x="241" y="212">203.0.113.7</text>
      <text x="399" y="212">203.0.113.7</text>
      <text x="553" y="212">203.0.113.7</text>
    </g>
  </g>

  <text x="320" y="255" text-anchor="middle" fill="currentColor" font-size="11.5" opacity="0.85">Every POP advertises the identical IP — BGP delivers each user's packets to the closest one.</text>
  <text x="320" y="274" text-anchor="middle" fill="currentColor" font-size="11.5" opacity="0.85">"One IP" quietly resolves to four different datacenters.</text>
</svg>

Anycast is the technique behind 1.1.1.1, every major CDN, and most global load balancers. Two consequences worth mentioning in an interview:

- **Failover is automatic.** If a datacenter withdraws its BGP route, traffic naturally drains to the next closest one.
- **Stateful protocols need care.** A long-lived TCP connection can rehome mid-session if BGP changes. Anycast plays best with stateless or short-lived flows; long-lived flows often pin clients to a region after the first packet.

## What this means for system design

Two patterns come up repeatedly:

**Global routing via DNS.** You use GeoDNS or latency-based DNS to send users to the nearest region. Pro: cheap, works everywhere. Con: TTL caching means failover is slow and uneven across the user base. Mitigation: keep TTLs low *and* assume some clients will be slow to follow.

**Global routing via Anycast.** You hand out a single IP, BGP does the work. Pro: instant convergence on failure. Con: requires owning IP space and BGP-capable infrastructure (or paying a provider that does). Mitigation: most teams use a managed edge product (Cloudflare, Fastly, AWS Global Accelerator) rather than rolling their own.

You will rarely build either from scratch in an interview, but you should be able to say *"we'll terminate TLS at the edge via Anycast, then route to the nearest healthy region; if a region fails, BGP takes traffic to the next one within seconds, with DNS as a slower secondary failover."*

## Why DNS is fragile

DNS is famously the cause of more outages than people expect. Three failure modes worth knowing:

- **Misconfigured TTLs.** A long TTL on a record you need to change locks you out of fast failover. A short TTL on a heavily-queried record pushes load onto your resolvers.
- **Negative caching.** Failed lookups (NXDOMAIN) are also cached. A typo in a record can break clients for the duration of the negative-cache TTL.
- **Provider outages.** DNS providers themselves have outages (a famous 2016 Dyn incident took down half the internet's name resolution for hours). For critical systems, use **multiple DNS providers** with the same records.

## What to say in an interview

If the prompt is a globally distributed service, DNS earns one or two minutes of attention:

> *"Clients resolve `api.example.com` against our DNS, which returns an Anycast IP that lands at the nearest edge POP. The edge terminates TLS and forwards to the closest healthy region. Failover between regions is BGP-based with seconds-level convergence; we also keep DNS TTLs low (60s) as a secondary mechanism for clients that bypass the edge."*

If the prompt is single-region, you can skip DNS entirely after a one-liner. Knowing when to spend time here, and when not to, is itself a signal.
