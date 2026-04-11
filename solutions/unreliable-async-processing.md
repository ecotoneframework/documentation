---
description: How to build reliable async processing in Laravel and Symfony with Ecotone
---

# Unreliable Async Processing

## The Problem You Recognize

You added async processing to handle background work — sending emails, processing payments, syncing data. But now you have new problems:

- Failed jobs **disappear silently** or retry forever with no visibility
- You **can't replay** a failed message after fixing the bug — the data is gone
- A **duplicate webhook** triggers the same handler twice, leading to double charges or duplicate emails
- Going async **required touching every handler** — adding queue configuration, serialization, and retry logic to each one

In **Laravel**, you've scattered `dispatch()` calls and `ShouldQueue` implementations across your codebase. In **Symfony**, you've configured Messenger transports and retry strategies in YAML, but each handler still needs custom error handling.

## What the Industry Calls It

**Resilient Messaging** — a combination of patterns: automatic retries, error channels, dead letter queues, the outbox pattern for guaranteed delivery, and idempotency for deduplication.

## How Ecotone Solves It

With Ecotone, making a handler async is a single attribute. Resilience is built into the messaging layer — not bolted on per handler:

```php
// Make any handler async with one attribute
#[Asynchronous("notifications")]
#[EventHandler]
public function sendWelcomeEmail(UserRegistered $event): void
{
    // If this fails, Ecotone retries automatically
    // If it keeps failing, it goes to the dead letter queue
    // You can replay it after fixing the bug
}
```

Retries, error channels, and dead letter queues are configured once at the channel level — every handler on that channel gets production resilience automatically:

```php
// Deduplication prevents double-processing
#[ServiceContext]
public function messagingConfiguration(): array
{
    return [
        // All handlers on "notifications" channel get these settings
        SimpleMessageChannelBuilder::createQueueChannel("notifications"),
    ];
}
```

## Next Steps

- [Asynchronous Handling](../modelling/asynchronous-handling/) — Make handlers async with a single attribute
- [Retries](../modelling/recovering-tracing-and-monitoring/resiliency/retries.md) — Configure automatic retry strategies
- [Error Channel and Dead Letter](../modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/) — Store failed messages for replay
- [Outbox Pattern](../modelling/recovering-tracing-and-monitoring/resiliency/outbox-pattern.md) — Guarantee message delivery
- [Idempotency (Deduplication)](../modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md) — Prevent double-processing

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Command Bus Instant Retries](../modelling/recovering-tracing-and-monitoring/resiliency/retries.md#customized-instant-retries) for synchronous commands, [Command Bus Error Channel](../modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/#command-bus-error-channel) for centralized error routing, and [Gateway-Level Deduplication](../modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md#deduplication-with-command-bus) that protects every handler behind a bus automatically.
{% endhint %}
