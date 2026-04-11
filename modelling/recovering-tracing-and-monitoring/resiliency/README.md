# Resiliency

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

A failed HTTP call crashes your handler. A duplicate webhook triggers double-processing. You've wrapped handlers in try/catch blocks and retry loops — each one slightly different. Error handling is scattered across your codebase with no consistent strategy.

## How Ecotone Solves It

Ecotone handles failures at the **messaging layer** — not per feature. Automatic retries, error channels, dead letter queues, the outbox pattern, and idempotency are configured once and apply to all handlers on a channel. When something fails, messages are preserved and can be replayed after the bug is fixed.

---

Explore the resiliency features:

- [Retries](retries.md) — Automatic retry strategies for transient failures
- [Error Channel and Dead Letter](error-channel-and-dead-letter/) — Store failed messages for later replay
- [Final Failure Strategy](final-failure-strategy.md) — What happens when all retries are exhausted
- [Idempotency (Deduplication)](idempotent-consumer-deduplication.md) — Prevent double-processing
- [Resilient Sending](resilient-sending.md) — Guaranteed delivery to async channels
- [Outbox Pattern](outbox-pattern.md) — Atomic message publishing with database transactions
- [Concurrency Handling](concurrency-handling.md) — Optimistic and pessimistic locking
