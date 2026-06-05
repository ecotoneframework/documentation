---
description: The delivery, ordering, idempotency, and consistency guarantees Ecotone provides
---

# Delivery Semantics & Guarantees

This page collects, in one place, the guarantees Ecotone provides — so you don't have to infer them from individual feature pages.

## Message delivery: at-least-once

Asynchronous channels deliver **at-least-once**. If a consumer crashes, or a handler throws after the message was taken from the channel, the message is redelivered (subject to the configured [retry strategy](resiliency/retries.md)). A message is therefore never silently lost — but a handler may see the same message more than once, which is why idempotency matters (see below).

## The outbox: atomic, single-storage

When you use the [DBAL / outbox channel](resiliency/outbox-pattern.md), the outgoing message is written **in the same database transaction** as your business state. There is no two-storage atomicity problem because there is only one storage — if the transaction commits, the message is stored; if it rolls back, neither the state change nor the message is persisted. Dispatch from the outbox to the broker then happens at-least-once.

## Idempotency: effectively-once within a retention window

Because delivery is at-least-once, consumers must tolerate duplicates. Ecotone's [deduplication](resiliency/idempotent-consumer-deduplication.md) does this for you: a redelivered message carrying the same deduplication key is detected and skipped before your handler runs.

This is **effectively-once within the deduplication store's retention window — not exactly-once**. A redelivery that arrives after the key has expired from the store will be processed again. Keep handlers idempotent for anything that must never double-apply.

## Ordering

* **Within a single aggregate / event stream:** ordering is guaranteed. Each event's version is strictly `previous + 1`, enforced by the Event Store's optimistic concurrency check.
* **Across aggregates, across handlers, and for splitter outputs:** no global ordering is guaranteed. Each message is retried and processed independently, so two messages on the same channel may be handled out of order. When you need ordering at scale, partition by aggregate — see [Gap Detection and Consistency](../event-sourcing/setting-up-projections/gap-detection-and-consistency.md).

## Read Model consistency

* A **synchronous** projection that writes through the **same DBAL connection** as the Event Store updates in the same transaction as the events — the Read Model is always consistent with the Event Stream.
* A **synchronous** projection that writes to a different connection or datastore, or any **asynchronous** projection, is **eventually consistent**: it catches up by reading from the Event Store starting at its last committed position.

## Per-handler failure isolation

A copy of each message is dispatched to every subscribing handler. Each handler retries and fails independently — one failing subscriber never aborts or re-runs the others. See [Message Handling Isolation](message-handling-isolation.md).
