---
title: Communication Protocols
description: REST, gRPC, GraphQL, WebSockets, and Server-Sent Events — when to pick each, what the trade-offs are, and what to say in the interview.
---

"Which protocol?" is a question that comes up in nearly every system design interview, usually around the API design step. Most candidates default to REST and never justify it. Strong candidates pick deliberately and explain why.

This page covers the five protocols you should know cold.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 260" role="img" aria-label="Protocol comparison: payload size, streaming, direction" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:12px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">The five protocols at a glance</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="20" y="50" width="120" height="160" rx="10"/>
    <rect x="150" y="50" width="120" height="160" rx="10" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="280" y="50" width="120" height="160" rx="10"/>
    <rect x="410" y="50" width="120" height="160" rx="10"/>
    <rect x="540" y="50" width="80" height="160" rx="10"/>
  </g>
  <g fill="currentColor" text-anchor="middle">
    <text x="80" y="75" font-weight="700">REST</text>
    <text x="80" y="95" font-size="11">HTTP + JSON</text>
    <text x="80" y="115" font-size="11">cacheable</text>
    <text x="80" y="135" font-size="11">verbose</text>
    <text x="80" y="155" font-size="11" opacity="0.8">public API</text>
    <text x="80" y="190" font-size="11" font-weight="600" opacity="0.85">→ default</text>
    <text x="210" y="75" font-weight="700">gRPC</text>
    <text x="210" y="95" font-size="11">HTTP/2 + protobuf</text>
    <text x="210" y="115" font-size="11">streaming</text>
    <text x="210" y="135" font-size="11">5–10× smaller</text>
    <text x="210" y="155" font-size="11" opacity="0.8">internal</text>
    <text x="210" y="190" font-size="11" font-weight="600" opacity="0.85">→ service-to-service</text>
    <text x="340" y="75" font-weight="700">GraphQL</text>
    <text x="340" y="95" font-size="11">flexible queries</text>
    <text x="340" y="115" font-size="11">strong schema</text>
    <text x="340" y="135" font-size="11">cache-tricky</text>
    <text x="340" y="155" font-size="11" opacity="0.8">multi-client UI</text>
    <text x="340" y="190" font-size="11" font-weight="600" opacity="0.85">→ varied clients</text>
    <text x="470" y="75" font-weight="700">WebSocket</text>
    <text x="470" y="95" font-size="11">bidirectional</text>
    <text x="470" y="115" font-size="11">long-lived TCP</text>
    <text x="470" y="135" font-size="11">stateful</text>
    <text x="470" y="155" font-size="11" opacity="0.8">chat / games</text>
    <text x="470" y="190" font-size="11" font-weight="600" opacity="0.85">→ real-time both-ways</text>
    <text x="580" y="75" font-weight="700">SSE</text>
    <text x="580" y="95" font-size="11">HTTP stream</text>
    <text x="580" y="115" font-size="11">server→client</text>
    <text x="580" y="135" font-size="11">simple</text>
    <text x="580" y="155" font-size="11" opacity="0.8">notifications</text>
    <text x="580" y="190" font-size="11" font-weight="600" opacity="0.85">→ push only</text>
  </g>
</svg>

## HTTP/REST

Plain HTTP, JSON bodies, resource-shaped URLs, standard methods (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`). The lingua franca of web APIs.

**Strengths**

- Universally understood; works through every proxy, gateway, and CDN.
- Easy to cache (`GET`s are cacheable by default).
- Trivial to debug (curl, browser devtools).
- Massive tooling ecosystem.

**Weaknesses**

- Verbose on the wire (JSON, headers).
- No native streaming.
- No schema enforcement out of the box — clients and servers drift unless you bolt on OpenAPI/JSON Schema.
- Each call is a round trip; chatty UIs amplify latency.

**Pick REST when** you have a public API, a wide variety of clients, simple request/response semantics, and you value tooling and cacheability over raw efficiency. It is the right default 70–80% of the time.

## gRPC

Google's RPC framework. HTTP/2 transport, Protocol Buffers (binary) payloads, schema-first via `.proto` files, code generation in every major language.

**Strengths**

- 5–10x smaller payloads than JSON for the same content.
- Built-in streaming (server-streaming, client-streaming, bidirectional).
- Strong contracts via protobuf schemas; backwards compatibility is explicit.
- HTTP/2 multiplexing — many concurrent calls over one TCP connection.
- Deadlines and cancellation propagate through the call graph.

**Weaknesses**

- Not browser-native (you need gRPC-Web with a translating proxy).
- Harder to debug and cache through standard infrastructure.
- Code generation is a build step you have to maintain.
- Less discoverable than REST — you need the `.proto` to know the API.

**Pick gRPC when** you control both ends, performance matters, and the API surface is internal — service-to-service in a polyglot microservices system is its sweet spot.

## GraphQL

A query language that lets clients ask for exactly the data they need from a typed schema. Implemented as a single `POST /graphql` endpoint in most setups.

**Strengths**

- Eliminates over- and under-fetching; one round trip can satisfy a complex screen.
- Strongly typed schema; great IDE support.
- Excellent for **client diversity** — mobile, web, and partner each ask for different shapes against the same backend.
- Built-in subscriptions for real-time updates.

**Weaknesses**

- Caching is harder than REST (it's a `POST` by default; query shape varies).
- N+1 queries are easy to write and a pain to debug; you need DataLoader-style batching.
- Authorization happens at the field level — easy to get wrong.
- Operational complexity: query cost analysis, depth limits, persisted queries.

**Pick GraphQL when** clients have varied needs against a complex domain, you control both the schema and the clients, and the team has the discipline to manage query complexity. It is *not* a default — it is a deliberate choice with real costs.

## WebSockets

A protocol upgrade from HTTP that gives you a single, long-lived, full-duplex TCP connection between client and server. Either side can push at any time.

**Strengths**

- Real-time bidirectional communication with minimal framing overhead.
- One connection per client instead of one per request.
- Browser-native.

**Weaknesses**

- Stateful — the server must keep a connection per client. Scales by connection count, not request count.
- Doesn't go through standard HTTP caches.
- Reconnect, backpressure, and message ordering are your problem.
- Long-lived connections complicate deploys (drain time) and load balancing.

**Pick WebSockets when** you need genuine bidirectional, low-latency messaging — chat, live collaboration, multiplayer games, trading dashboards. For "the server pushes notifications occasionally," Server-Sent Events are simpler.

## Server-Sent Events (SSE)

A simple HTTP-based protocol where the server streams events to the client over a long-lived `GET` connection. One-way: server to client only.

**Strengths**

- Just HTTP — works through every proxy, CDN, and load balancer.
- Auto-reconnect with last-event-ID built into the browser.
- Trivially easy to implement.
- Excellent for "server pushes updates, client reads" patterns.

**Weaknesses**

- One direction only — client-to-server still needs a separate request.
- One TCP connection per stream (mitigated by HTTP/2).
- Less widely known than WebSockets, occasionally surprising to operators.

**Pick SSE when** the data flow is mostly server-to-client (notifications, live feeds, progress events, log streams) and you don't want the operational weight of WebSockets.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 198" role="img" aria-label="WebSocket vs SSE: one TCP connection, full duplex vs server-only" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <defs>
    <marker id="cp-ah" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="11" markerHeight="11" markerUnits="userSpaceOnUse" orient="auto">
      <path d="M0,1 L9,5 L0,9 z" fill="currentColor"/>
    </marker>
  </defs>
  <text x="320" y="24" text-anchor="middle" fill="currentColor" font-weight="600">WebSocket vs Server-Sent Events</text>

  <g transform="translate(0,0)">
    <text x="161" y="54" text-anchor="middle" fill="currentColor" font-weight="600">WebSocket — bidirectional</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="40" y="70" width="86" height="58" rx="8"/>
      <rect x="196" y="70" width="86" height="58" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="12">
      <text x="83" y="103">Client</text>
      <text x="239" y="103" font-weight="600">Server</text>
    </g>
    <g stroke="currentColor" stroke-width="2" fill="none">
      <path d="M126 90 H194" marker-end="url(#cp-ah)"/>
      <path d="M196 108 H128" marker-end="url(#cp-ah)"/>
    </g>
    <text x="161" y="160" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">One long-lived TCP — either side can push.</text>
    <text x="161" y="178" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Use: chat, games, live collaboration.</text>
  </g>

  <g transform="translate(320,0)">
    <text x="161" y="54" text-anchor="middle" fill="currentColor" font-weight="600">SSE — server only</text>
    <g fill="none" stroke="currentColor" stroke-width="2">
      <rect x="40" y="70" width="86" height="58" rx="8"/>
      <rect x="196" y="70" width="86" height="58" rx="8" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="12">
      <text x="83" y="103">Client</text>
      <text x="239" y="103" font-weight="600">Server</text>
    </g>
    <path d="M126 90 H194" stroke="currentColor" stroke-width="2" fill="none" stroke-dasharray="4 3" opacity="0.5" marker-end="url(#cp-ah)"/>
    <path d="M196 108 H128" stroke="currentColor" stroke-width="2" fill="none" marker-end="url(#cp-ah)"/>
    <text x="161" y="160" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Dashed = initial request; server then streams events.</text>
    <text x="161" y="178" text-anchor="middle" fill="currentColor" font-size="11" opacity="0.85">Use: notifications, progress, live feeds.</text>
  </g>
</svg>

## A decision cheatsheet

| If… | Use… |
|-----|------|
| Public API for unknown clients | REST |
| Internal service-to-service, polyglot | gRPC |
| Client UI needs flexible queries against one schema | GraphQL |
| Bidirectional real-time (chat, multiplayer) | WebSockets |
| Server pushes updates, client mostly listens | SSE |
| File upload / download | REST with multipart or signed URLs |
| Live video / audio | WebRTC (out of scope here) |

You can combine these in a single system. A common modern shape is REST or gRPC for normal operations + WebSockets or SSE for real-time updates + signed URLs for media.

## Common pitfalls

**Picking gRPC for public APIs.** Browsers can't speak it natively, partners hate generating clients, and the perceived performance win disappears when you add a translating proxy. Use REST or GraphQL externally; gRPC internally.

**Picking GraphQL because it sounds modern.** Mobile teams especially burn weeks on N+1 queries, caching, and query budgets. If your client needs are simple, REST is shorter and faster.

**Using WebSockets for occasional pushes.** SSE or simple long-polling is usually cheaper to operate.

**Forgetting versioning.** Every protocol needs a story: REST uses URL versions or media-type versions; gRPC uses proto evolution rules; GraphQL deprecates fields. Mention versioning when you mention the protocol.

## What to say in an interview

A solid one-liner that covers the common case:

> *"The public API is REST over HTTPS — easy for clients, cacheable, and we don't need streaming there. Internally the services talk gRPC over HTTP/2 because of the smaller payloads and built-in deadlines. For real-time updates to the client we add an SSE channel, and we keep WebSockets in reserve if we ever need bidirectional messaging."*

Three protocols, three reasons. That is the level of specificity that earns credit.
