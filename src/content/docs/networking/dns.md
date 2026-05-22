---
title: DNS & The Client-Server Model
description: How a request finds your servers — DNS resolution, the role of TTLs, Anycast, and the parts of the client-server model that matter in interviews.
---

Before any of your servers see a packet, a client has to figure out *where to send it*. That work happens at the very edge — and it shows up in system design interviews more often than candidates expect, especially when the question turns to global scale, multi-region failover, or CDN strategy.

## The client-server model in one paragraph

A client (browser, mobile app, another service) wants a resource. It opens a connection to a server identified by an IP address, sends a request, and gets a response. Between the client and the server there may be a load balancer, an API gateway, a CDN, a service mesh, and several layers of caching. Every box in your interview diagram exists either to *resolve, route, or serve* requests faster, cheaper, or more reliably than a direct connection would.

The key idea is that the system is **fundamentally pull-based**: the client initiates and the server responds. Notifications, websockets, and server-sent events bend this rule but don't break it — the connection is still established by the client.

<img src="/diagrams/networking/dns-1.svg" alt="DNS resolution chain from browser to authoritative nameserver" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

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

<img src="/diagrams/networking/dns-2.svg" alt="Anycast: four users each routed to their nearest POP, all advertising the same IP" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;"/>

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
