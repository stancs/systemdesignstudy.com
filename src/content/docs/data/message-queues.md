---
title: Message Queues & Streams
description: Queues vs streams, delivery guarantees, ordering, retries and dead-letter queues, and how to pick between Kafka, RabbitMQ, SQS, and Pub/Sub.
---

A message queue (or stream) is the way distributed systems do asynchronous, decoupled communication. Producers write messages; consumers read them, often on a different schedule, often on different machines, sometimes hours or days later. The queue absorbs spikes, smooths bursts, and lets services evolve independently.

Almost every non-trivial system design has at least one queue or stream in it. Knowing the difference between them — and what guarantees you actually get — is one of the highest-yield areas to study.

## Queue vs stream

The two words get used interchangeably; the underlying models are genuinely different.

**Queue (task queue).** A message is delivered to **one consumer** out of a worker pool, processed, and then removed. Think "to-do list." Examples: RabbitMQ (classic mode), AWS SQS, Google Cloud Tasks, Celery.

**Stream (event log).** A message is **persisted in an append-only log** and any number of consumers can read it independently, each tracking its own position. Think "audit log other systems can replay." Examples: Apache Kafka, AWS Kinesis, Google Pub/Sub, Apache Pulsar.

Two simple rules of thumb:

- If each message represents *work that must be done exactly once by one worker*, you want a queue.
- If each message represents *an event that multiple systems care about*, you want a stream.

You can fake one with the other, but the natural shape of the workload usually points at the right answer.

## Delivery guarantees

Three guarantees you should know cold. There are no others; every system implements one of these.

**At-most-once.** Messages may be lost; they're never delivered twice. Cheapest and simplest. Acceptable for metrics, log shipping, ephemeral notifications.

**At-least-once.** Messages are never lost; they may be delivered more than once. The default in nearly every queue and stream. Forces consumers to be **idempotent** — processing the same message twice must produce the same result as processing it once.

**Exactly-once.** Each message is processed exactly once. Possible in narrow cases (Kafka has it within the Kafka ecosystem with transactional producers + idempotent consumers; Pulsar similarly). Genuine end-to-end exactly-once that includes external side effects (sending an email, charging a card) is a myth — what you actually do is at-least-once + idempotent processing.

In an interview, the right answer is almost always *"at-least-once delivery, idempotent consumers."* If you say "exactly-once" without qualification, be ready to defend exactly how.

## Ordering

A few flavors:

- **No ordering.** Messages can arrive in any order. Cheapest, scales infinitely. Pick when the consumer doesn't care (e.g., metrics).
- **Per-key ordering.** Messages with the same key arrive in the order they were produced. The most common shape. Kafka guarantees this within a partition; SQS FIFO guarantees it within a message group; Pulsar within a key.
- **Strict global ordering.** All messages arrive in the order produced, across all keys. Effectively only achievable by funneling everything through one partition or one writer. Doesn't scale.

If your design needs ordering, it almost always wants the **per-key** flavor. State the key explicitly: *"ordering is guaranteed per user_id; cross-user events have no global order."*

## Idempotency, retries, and DLQs

Three patterns that almost every queue-based design needs.

**Idempotency.** A consumer can process the same message twice without breaking. Achieve via:

- A unique message ID + a "seen messages" table (TTL-bounded).
- Conditional writes (`INSERT ... ON CONFLICT DO NOTHING`).
- Deriving outcomes from immutable inputs only.

Even if your queue claims exactly-once, build idempotency. Network is unreliable; consumers crash; retries happen.

**Retries.** When processing fails, the message goes back on the queue (or its visibility timeout expires and it reappears). Standard practice: **exponential backoff with jitter** between retries to avoid thundering herds. After N failures, give up and route the message to a dead-letter queue.

**Dead-letter queue (DLQ).** A holding area for messages that couldn't be processed. Critical for debugging: instead of looping forever or silently dropping them, they go somewhere you can inspect. Always have a DLQ. Always alert on DLQ depth.

## The systems by name

A quick scorecard for the four you'll most likely mention:

**Apache Kafka.** Distributed append-only log. Massive throughput, per-partition ordering, configurable retention (often days or weeks), excellent for event streaming and replayable analytics. Higher operational complexity than queues; pick a managed offering if you can.

**RabbitMQ.** Classic message broker with rich routing (exchanges, bindings, topics). Excellent task-queue semantics. Lower throughput than Kafka; simpler operationally. Great for traditional work queues.

**AWS SQS.** Fully managed, pull-based, at-least-once standard queues + FIFO queues. Boring in the best way — pay per message, no operational burden. Good default for AWS-based work queues.

**Google Pub/Sub / AWS Kinesis.** Cloud-managed streams; conceptually similar to Kafka with less knob-twiddling. Reasonable choices when you don't want to operate Kafka yourself.

If the prompt is platform-agnostic, "we'll use Kafka for the event stream and SQS for the work queue" is fine and credible.

## Backpressure

Queues buffer producer/consumer mismatch — but the buffer is not infinite. When producers persistently outpace consumers, three things can happen:

- The queue grows without bound (until disk fills).
- Producers are slowed via **backpressure** (the queue tells them to slow down).
- Old messages get dropped or expire.

Mention backpressure when discussing queues at scale. The default behavior of most queues is to grow unbounded, which means **alerting on queue depth** is non-negotiable. Pair it with autoscaling on consumers, and decide explicitly what happens if you can't catch up — shed load, drop low-priority traffic, or page someone.

## Use cases that come up in interviews

**Decoupling slow work from request path.** User uploads a photo; the API returns immediately and enqueues a "resize this photo" job. Worker pool processes in the background.

**Smoothing spikes.** Black Friday surge of orders. Web tier accepts and enqueues; payments tier drains the queue at its own rate without crashing.

**Fan-out.** One event ("user signed up") needs to trigger ten downstream actions (welcome email, analytics, CRM, recommendation warmup, …). Each is its own consumer of the same stream.

**Event sourcing / audit log.** Every state change is written to the stream first; the database is rebuilt from the stream. Replayable, debuggable, painful to operate but powerful when the requirements fit.

**CDC (change data capture).** Database changes are streamed out so other systems (search indexes, caches, downstream services) can stay in sync. Debezium → Kafka is a common pattern.

## What to say in an interview

A clean way to introduce a queue:

> *"The write path enqueues a `ProcessOrder` message to Kafka, keyed by user_id so per-user ordering is preserved. A worker pool consumes from the topic, processes idempotently using the order_id as a dedupe key, and writes to the database. Failed messages retry with exponential backoff three times, then go to a DLQ that alerts on depth. The stream is retained for 7 days so we can replay if a downstream consumer needs a backfill."*

Five decisions — which system, partitioning key, idempotency, retry policy, retention — and a sentence each. That's the level of specificity that earns credit.
