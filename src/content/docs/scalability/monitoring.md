---
title: Monitoring & Observability
description: Metrics, logs, traces, and the SLO-driven mindset that turns "we have dashboards" into "we know when something is wrong before users do."
---

You can't operate a distributed system you can't see. Monitoring and observability are how you go from "the user filed a ticket" to "we paged ourselves about it 90 seconds before the user noticed" — and that's the bar you should aim for in any interview answer involving real-world operability.

The terms get conflated, so let's separate them.

## Monitoring vs observability

**Monitoring** is the practice of collecting predefined signals (CPU, requests/sec, error rate, queue depth) and alerting when they cross thresholds. You know what to look for.

**Observability** is the property of being able to **answer new questions about your system** without shipping new code. It's about having enough detail in your data — high-cardinality metrics, structured logs, distributed traces — that you can drill into a novel problem post-hoc.

Most teams have basic monitoring (dashboards, alerts). Fewer have real observability. In an interview, "we'll have monitoring" is a checkbox; "we'll instrument enough to debug unknown unknowns" is the senior version.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 270" role="img" aria-label="Metrics, logs, and traces — the three pillars of observability" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="24" text-anchor="middle" fill="currentColor" font-weight="600">The three pillars</text>
  <g transform="translate(20,44)">
    <rect width="190" height="206" rx="10" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2"/>
    <text x="95" y="32" text-anchor="middle" fill="currentColor" font-weight="700">Metrics</text>
    <polyline points="20,108 40,96 60,101 80,80 100,86 120,64 140,70 160,50 170,58" stroke="currentColor" stroke-width="1.5" fill="none"/>
    <g fill="currentColor" font-size="11">
      <text x="20" y="146">numbers over time</text>
      <text x="20" y="166">cheap, alertable</text>
      <text x="20" y="186">quantitative</text>
    </g>
  </g>
  <g transform="translate(225,44)">
    <rect width="190" height="206" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="95" y="32" text-anchor="middle" fill="currentColor" font-weight="700">Logs</text>
    <g stroke="currentColor" stroke-width="1.5" fill="none">
      <line x1="20" y1="62" x2="170" y2="62"/>
      <line x1="20" y1="78" x2="170" y2="78"/>
      <line x1="20" y1="94" x2="170" y2="94"/>
      <line x1="20" y1="110" x2="170" y2="110"/>
    </g>
    <g fill="currentColor" font-size="11">
      <text x="20" y="146">timestamped events</text>
      <text x="20" y="166">structured fields</text>
      <text x="20" y="186">qualitative</text>
    </g>
  </g>
  <g transform="translate(430,44)">
    <rect width="190" height="206" rx="10" fill="none" stroke="currentColor" stroke-width="2"/>
    <text x="95" y="32" text-anchor="middle" fill="currentColor" font-weight="700">Traces</text>
    <g fill="currentColor" opacity="0.5">
      <rect x="20" y="54" width="150" height="12" rx="3"/>
      <rect x="30" y="71" width="105" height="12" rx="3"/>
      <rect x="44" y="88" width="48" height="12" rx="3"/>
      <rect x="100" y="88" width="30" height="12" rx="3"/>
    </g>
    <g fill="currentColor" font-size="11">
      <text x="20" y="146">request waterfall</text>
      <text x="20" y="166">across services</text>
      <text x="20" y="186">latency analysis</text>
    </g>
  </g>
</svg>

## The three pillars

The conventional framing — useful even if a little reductive.

**Metrics.** Numbers over time. Cheap to store, cheap to query, easy to alert on. Best for *quantitative* questions: how many requests, how fast, how often errors. Examples: Prometheus, Datadog metrics, CloudWatch metrics.

**Logs.** Textual events with timestamps. Best for *qualitative* questions: what exactly happened in this request. Structured logs (JSON with consistent fields) are dramatically more useful than free-text. Examples: Elasticsearch/OpenSearch, Loki, Splunk, CloudWatch Logs.

**Traces.** End-to-end views of a single request as it travels through multiple services. Best for *latency and dependency* questions: which downstream service is slow, which dependency timed out, which path is the critical path. Examples: Jaeger, Zipkin, OpenTelemetry-based traces in Datadog/Honeycomb.

Modern best practice is to instrument with **OpenTelemetry** (vendor-neutral SDKs and protocols) and ship to whichever backends you prefer. Mentioning OpenTelemetry is enough to signal currency.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 240" role="img" aria-label="The four golden signals: latency, traffic, errors, saturation" style="max-width:100%;height:auto;margin:1.5rem auto;display:block;font:13px/1.3 ui-sans-serif,system-ui,sans-serif;color:inherit;">
  <text x="320" y="22" text-anchor="middle" fill="currentColor" font-weight="600">The four golden signals</text>
  <g fill="none" stroke="currentColor" stroke-width="2">
    <rect x="20" y="50" width="140" height="170" rx="10"/>
    <rect x="170" y="50" width="140" height="170" rx="10"/>
    <rect x="320" y="50" width="140" height="170" rx="10" fill="var(--sl-color-accent-low,#dbeafe)" stroke="var(--sl-color-accent,#3b82f6)"/>
    <rect x="470" y="50" width="150" height="170" rx="10"/>
  </g>
  <g fill="currentColor" text-anchor="middle">
    <text x="90" y="78" font-weight="700">Latency</text>
    <text x="90" y="98" font-size="11">how long</text>
    <text x="90" y="116" font-size="11">p50 / p95 / p99</text>
    <text x="240" y="78" font-weight="700">Traffic</text>
    <text x="240" y="98" font-size="11">how many</text>
    <text x="240" y="116" font-size="11">req / sec</text>
    <text x="390" y="78" font-weight="700">Errors</text>
    <text x="390" y="98" font-size="11">fraction failing</text>
    <text x="390" y="116" font-size="11">by status code</text>
    <text x="545" y="78" font-weight="700">Saturation</text>
    <text x="545" y="98" font-size="11">how full</text>
    <text x="545" y="116" font-size="11">CPU, queue, pool</text>
  </g>
  <g stroke="currentColor" stroke-width="1.5" fill="none">
    <polyline points="40,180 60,170 80,175 100,165 120,160 140,155" stroke-width="2"/>
    <polyline points="190,180 210,170 230,165 250,150 270,160 290,140" stroke-width="2"/>
    <polyline points="340,180 360,175 380,170 400,180 420,150 440,140" stroke="var(--sl-color-accent,#3b82f6)" stroke-width="2"/>
    <polyline points="490,190 510,185 530,180 550,170 570,160 600,155" stroke-width="2"/>
  </g>
</svg>

## The four golden signals

Originally Google SRE's framing for what to monitor on every service. Memorize:

- **Latency** — how long requests take. Track p50, p95, p99 separately (averages lie).
- **Traffic** — how many requests, per second.
- **Errors** — fraction of requests failing, by status code.
- **Saturation** — how full the system is (CPU, memory, queue depth, connection pool).

If you can dashboard those four for every service, you've covered 90% of operational visibility. RED (Rate, Errors, Duration) and USE (Utilization, Saturation, Errors) are similar acronyms describing the same idea from slightly different angles.

## Percentiles, not averages

This is one of the single most important habits to internalize. **Averages hide everything that matters.**

A service with a 100ms average latency might have a 50ms median and a 2-second p99. The average and the median are fine; the p99 means 1% of users wait two seconds. At a million requests, that's 10,000 unhappy users per query period.

Rules of thumb worth quoting in an interview:

- **Always alert on p99 or p99.9**, not p50 or average.
- **Latency budgets are end-to-end.** A 200ms p99 budget can't be met if you have ten serial downstream calls each at 50ms p99 — the tail amplifies.
- **Slowest links dominate.** Tail latency is set by your worst replica, not your average one.

## SLIs, SLOs, and error budgets

A more disciplined frame:

- **SLI** (service-level indicator). A measurable property of the service. "Fraction of requests returning within 200ms."
- **SLO** (service-level objective). A target value for the SLI. "99% of requests within 200ms over a rolling 30 days."
- **Error budget.** The complement of the SLO. "We're allowed to fail 1% of requests; how much have we burned?"

The point of an error budget isn't compliance — it's a **forcing function for prioritization**. If you've burned 80% of the budget two weeks into the month, the team focuses on reliability over features. If you've burned 5%, the team is free to ship riskier work.

In an interview, naming an SLO is much more impressive than saying "high availability." Saying *"99.95% of writes succeed within 1 second, measured at the gateway, over a rolling 28-day window"* sounds like an engineer who has owned a production service.

## High-cardinality data: where observability lives

The reason "metrics, logs, traces" isn't sufficient on its own is that metrics traditionally roll up to *low* cardinality (a few tags) for cost reasons. You can ask "how many errors?" — you can't always ask "how many errors for users on Android 12 in São Paulo?"

The newer wave of observability tools (Honeycomb is the iconic example; Datadog and others follow) keeps **wide events** — every request becomes a row of dozens or hundreds of attributes — and queries across them at high cardinality. That's how you debug "why are exactly these 17 users failing?"

You don't need to use any specific product. You need to **structure your logs and traces with enough dimensions** to answer questions you didn't anticipate. Things like user_id, request_id, deployment_version, region, feature_flag — add them as fields, not by parsing free text.

## Distributed tracing in one paragraph

Each incoming request gets a unique **trace ID**, propagated through every downstream call. Each service emits a **span** describing the work it did, parent-linked to the calling span. The collection of spans for one trace ID reconstructs the request's path through the system. You can then see, in a flame graph, which span is the slow one.

This is the only practical way to debug latency in a microservices system. Without it, you're guessing.

Two practical notes:

- **Sample.** Tracing every request is expensive. Tail-based sampling (keep all spans for slow or errored requests, sample the rest) is the modern approach.
- **Propagate trace context.** Use W3C `traceparent` headers. Everything else is bespoke and brittle.

## Alerts: the part everyone gets wrong

The two biggest alerting mistakes:

- **Alerting on causes, not symptoms.** You don't care that CPU is 90%; you care that latency is bad and errors are up. Alert on the user-facing symptom; investigate the cause from there.
- **Too many alerts.** Every false alarm trains the on-call to ignore alerts. Aim for **every alert is actionable**. If an alert fires and the answer is "wait, it'll clear up," delete the alert.

A widely used heuristic: alerting should be **SLO-burn-rate based**. Page when you're going to miss the SLO; don't page on every threshold cross. Google SRE's "Alerting on SLOs" chapter is the canonical reference.

## What to instrument first

If you're building a new service, in order:

1. **The four golden signals**, exposed as Prometheus-style metrics.
2. **Structured logs**, with request_id, user_id, route, status, latency, error message.
3. **Distributed tracing**, propagating W3C trace context.
4. **A health endpoint** for liveness/readiness probes.
5. **An alerting plan** tied to SLOs, not raw thresholds.
6. **A dashboard per service** displaying the four signals over the last hour.

Mention as many of these as fit the prompt. Six bullets is too many; pick the two or three the interviewer most wants to hear.

## Common pitfalls

**Alert spam.** Page fatigue is a real outage cause. Tune ruthlessly.

**Metrics without context.** A graph that says "errors are up" with no way to know which endpoint, which version, which user segment — useless during an incident. Tag your metrics.

**Logs as your primary debugging tool.** Logs are essential, but searching them as your default tool means every incident is a 30-minute archaeology session. Metrics for "what's wrong" + traces for "where" + logs for "exactly what" is the modern split.

**No way to test instrumentation.** If you haven't run a fire drill, your monitoring probably doesn't work the way you think. Run synthetic incidents periodically.

## What to say in an interview

A clean observability paragraph:

> *"Every service emits the four golden signals as Prometheus metrics — latency p50/p95/p99, traffic, errors, and saturation. Logs are structured JSON with request_id and user_id on every line. We use OpenTelemetry for distributed tracing with tail-based sampling at 1% of normal traffic and 100% of slow or errored requests. SLO is 99.95% of writes succeed in under 1 second over a 28-day window, and we alert on burn rate — pages fire when we're going to miss the SLO, not on every spike. Every alert has a runbook."_*

Eight specific decisions in five sentences. That's how observability is graded.
