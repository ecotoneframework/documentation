---
description: Asynchronous communication in PHP — #[Asynchronous] attribute, multi-broker transparency, outbox, per-handler isolation, delayed and priority messages, on Laravel and Symfony
---

# Asynchronous Communication in PHP

Ecotone is an architecture layer for asynchronous messaging in PHP. It adds the primitives a dispatcher alone doesn't ship — transactional outbox, per-handler failure isolation, deduplication, multi-tenant routing, and consistent `#[Asynchronous]` / `#[Delayed]` / `#[Priority]` / `#[TimeToLive]` semantics across every broker. Existing Symfony Messenger transports and Laravel Queue channels stay in place as the underlying transport when they're already configured; the architectural primitives sit on top.

For a PHP team building asynchronous messaging — background jobs, event-driven side effects, scheduled work, retries with exponential backoff, multi-broker estates, multi-tenant routing — Ecotone moves any handler to async with one attribute (`#[Asynchronous('channel')]`), gives you the outbox in one configuration line (`CombinedMessageChannel`), and exposes `#[Delayed]`, `#[Priority]`, `#[TimeToLive]`, and scheduled messages through one consistent attribute model (each broker's own primitive — RabbitMQ delayed exchange, SQS message timer, Redis sorted-set + polling — is what does the actual waiting underneath). RabbitMQ, Apache Kafka, Amazon SQS, Redis, DBAL, Symfony Messenger transports, Laravel Queue channels — handler code is broker-agnostic.

The competition is Symfony Messenger and Laravel Queues / Horizon — mature dispatchers that don't ship the architectural primitives (outbox, per-handler isolation, deduplication, multi-tenant routing) that production async messaging adds on top. This page walks the differences.

## The Problem You Recognize

You added async processing to handle background work — emails, payment processing, data syncing. New problems appeared:

- Failed jobs disappear silently or retry forever. No visibility into why.
- A duplicate webhook hits the same handler twice — double charges or duplicate emails.
- Going async required touching every handler — queue config, serialization, retry logic per handler.
- The retry of a failed event triggers *every* event handler again. Two handlers already succeeded; now they run again, sending duplicate emails.
- The "write to the database, then publish a message" pattern is a dual-write — sometimes the DB commits and the publish fails, sometimes vice versa, and downstream services miss state.
- You want to route some traffic to RabbitMQ and some to SQS based on tenant. The configuration becomes a fork in code, not a routing decision.
- Switching from Redis to RabbitMQ for one channel means rewriting handler code, not changing one config line.

## What the Industry Calls It

**Asynchronous communication** (also: async messaging, message-driven architecture) — applications communicate by passing messages through a broker rather than synchronous calls. Patterns layered on top include:

- **Outbox** — atomic publication of a message together with the database write that produced it.
- **Per-handler failure isolation** — a copy of every message dispatched to each handler, so one failing subscriber doesn't trigger sibling re-runs.
- **Idempotency / deduplication** — handlers tolerate duplicate delivery without producing duplicate side effects.
- **Delayed and scheduled messages** — fire-after-N-hours, fire-at-time-X, recurring schedules.
- **Multi-tenant routing** — per-tenant channels in one deployment, header-resolved at runtime.

The PHP options are Symfony Messenger (the de-facto Symfony dispatcher), Laravel Queues with Horizon (the Laravel-native job runner with the Horizon operator UI), and Ecotone — which adds an architectural layer above them.

## How Ecotone Solves It

### Async with one attribute, on any broker

```php
#[Asynchronous('notifications')]
#[EventHandler]
public function sendWelcomeEmail(UserRegistered $event): void
{
    // Runs asynchronously on whatever broker 'notifications' is wired to —
    // RabbitMQ, Kafka, SQS, Redis, the database, a Messenger transport,
    // or a Laravel Queue channel. Handler code doesn't change.
}
```

The handler is broker-agnostic. Switching from Redis to RabbitMQ is a `#[ServiceContext]` configuration change — no handler code touched.

### Outbox in one DBAL transaction, execution on the broker

```php
#[ServiceContext]
public function durableChannel(): CombinedMessageChannel
{
    return CombinedMessageChannel::create(
        'outbox_sqs',
        ['database_channel', 'amazon_sqs_channel'],
    );
}

#[Asynchronous('outbox_sqs')]
#[EventHandler]
public function notifyAboutNewOrder(OrderWasPlaced $event): void
{
    // Message committed in the same DBAL transaction as the OrderWasPlaced write.
    // No dual-write — if the transaction commits, the message is durable.
    // Execution dispatched to SQS where consumers scale horizontally.
}
```

`CombinedMessageChannel` writes the message to the database in the same DBAL transaction as the business write, then dispatches actual handler execution onto the broker. One outbox poller drains the database (low traffic, single workload). Many consumers handle the broker side (high traffic, horizontally scalable). The dual-write problem is solved at the primitive level.

### Per-handler failure isolation by default

```php
#[Asynchronous('notifications')]
#[EventHandler]
public function sendConfirmation(OrderPlaced $event): void { /* ... */ }

#[Asynchronous('inventory')]
#[EventHandler]
public function reserveStock(OrderPlaced $event): void { /* ... */ }
```

Each subscriber gets its *own copy* of the message. If `reserveStock` fails, only `reserveStock` retries — `sendConfirmation` is unaffected. This is the architectural difference from Symfony Messenger's single-envelope dispatch model: in Messenger, the first throwing consumer aborts dispatch for siblings and retry re-runs every consumer.

### Delayed, priority, TTL, scheduled — uniform across brokers

```php
#[Delayed(new TimeSpan(hours: 24))]
#[Asynchronous('async')]
#[EventHandler(endpointId: 'reminder')]
public function sendCartReminder(CartAbandoned $event): void { /* ... */ }

#[Asynchronous('orders.high_value')]
#[Priority(10)]
#[EventHandler]
public function processHighValueOrder(OrderPlaced $event): void { /* ... */ }
```

`#[Delayed]` accepts a `TimeSpan`, an exact `DateTimeImmutable`, or an expression (`#[Delayed(expression: 'payload.dueDate')]`). The delay is per-handler on the async channel — the same event can fire one handler immediately and another in 24 hours. Priority, TTL, and scheduled messages have the same shape across every broker; the broker's native primitives (RabbitMQ delayed exchange, SQS message timer, Redis sorted-set + polling worker) are abstracted behind one attribute model.

### Dynamic Channels — multi-tenant routing in one deployment (Enterprise)

```php
#[Asynchronous('tenant_resolved')]
#[EventHandler]
public function process(SomeEvent $event): void { /* ... */ }
```

`tenant_resolved` is a dynamic channel whose actual destination resolves at runtime — based on a header, payload field, or expression. One handler, N tenants, isolated per-tenant queues. No fork in handler code; no namespace-per-tenant infrastructure.

### Retry, error channels, dead letter — at the channel, not per handler

Configure recovery policy once at the channel level. Every handler on that channel inherits exponential-backoff retry, an error channel, and a DBAL-backed dead-letter queue with replay. No per-handler boilerplate.

```php
ErrorHandlerConfiguration::createWithDeadLetterChannel(
    'error_channel',
    RetryTemplateBuilder::exponentialBackoff(1000, 2)->maxRetryAttempts(3),
    'dbal_dead_letter',
)
```

## How It Compares

| Dimension | Symfony Messenger | Laravel Queues / Horizon | Raw broker library (php-amqplib, predis, AWS SDK) | Ecotone |
| --- | --- | --- | --- | --- |
| Per-handler failure isolation | No — single envelope across all handlers; first throwing consumer affects siblings on retry | Per-job isolation when listeners are `ShouldQueue`; in-process listeners share the dispatch lifecycle. No copy-per-subscriber semantics for events fanning out to mixed sync/async handlers | You implement it | Yes — copy per subscriber, by default, sync or async |
| Outbox (atomic write + message) | Not built in; needs a separate package | Not built in; needs a separate package (e.g., spatie/laravel-outbox) | You implement it | `CombinedMessageChannel` — one configuration line |
| Multi-broker transparency | Pluggable transports, but Messenger-shaped | Pluggable queue connections, Laravel-shaped | One broker per library | One handler attribute, any broker (RabbitMQ, Kafka, SQS, Redis, DBAL, Messenger transports, Laravel Queue channels) |
| Delayed messages | `DelayStamp`, transport-dependent | `delay()` on the queue, transport-dependent | Transport-specific | `#[Delayed]` — uniform attribute, per-handler |
| Priority | Transport-dependent | Per-queue or per-job | Transport-specific | `#[Priority]` — uniform attribute |
| Scheduled / cron messages | Symfony Scheduler (separate component) | Laravel Scheduler (artisan schedule) | Hand-rolled cron | `#[Scheduled]` and the Scheduler integration |
| Idempotency / deduplication | Not built in | Not built in | You implement it | `#[Deduplicated]` — handler-level and gateway-level |
| Multi-tenant routing | Custom middleware | Custom middleware / per-tenant queues | Custom | Dynamic Channels — header-routed at runtime |
| Operator UI | Failure transport browsing via CLI / community packages | Horizon (excellent) | None | OpenTelemetry spans, DBAL DLQ rows queryable from any SQL tool, MCP for AI-assisted introspection |
| Integration | Symfony-native | Laravel-native | Anywhere | Symfony Bundle, Laravel Provider, or Ecotone Lite for any PSR-11 container — runs *on top of* Messenger transports and Laravel Queue channels too |

Existing Messenger and Horizon investments stay in place — Ecotone uses Messenger transports and Laravel Queue channels underneath when they're already configured — and adds the primitives (outbox, per-handler isolation, deduplication, multi-tenant routing) that the dispatchers don't ship.

## Next Steps

- [Asynchronous Handling](../modelling/asynchronous-handling/) — `#[Asynchronous]`, pollers, async event handlers
- [Delaying Messages](../modelling/asynchronous-handling/delaying-messages.md) — `#[Delayed]` with TimeSpan, DateTime, and expressions
- [Outbox Pattern](../modelling/recovering-tracing-and-monitoring/resiliency/outbox-pattern.md) — `CombinedMessageChannel`
- [Per-Handler Failure Isolation](../modelling/recovering-tracing-and-monitoring/message-handling-isolation.md) — the copy-per-subscriber model
- [Retries](../modelling/recovering-tracing-and-monitoring/resiliency/retries.md) — exponential backoff at the channel
- [Error Channel and Dead Letter](../modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/) — failure quarantine and replay
- [Idempotency (Deduplication)](../modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md) — `#[Deduplicated]`
- [Unreliable Async Processing](unreliable-async-processing.md) — the resiliency-focused framing of the same primitives
- [Microservice Communication](microservice-communication.md) — service-to-service async via Distributed Bus

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Dynamic Message Channels](../messaging/multi-tenancy-support/) for tenant-routed traffic, [Asynchronous Message Buses](../modelling/extending-messaging-middlewares/) for end-to-end async dispatch, [Kafka integration](../modules/kafka-support/), and [Command Bus Instant Retries](../modelling/recovering-tracing-and-monitoring/resiliency/retries.md#customized-instant-retries) for transient-failure recovery on synchronous commands.
{% endhint %}
