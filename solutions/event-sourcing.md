---
description: Event Sourcing in PHP — built-in event store, partitioned and streaming projections, gap detection, projection emission, and end-to-end PII encryption on Laravel and Symfony
---

# Event Sourcing in PHP

For a PHP team building event-sourced systems, Ecotone delivers a built-in event store on PostgreSQL or MySQL, `#[EventSourcingAggregate]` for event-sourced aggregates, and `#[ProjectionV2]` for read models — global tracked, partitioned, or streaming. Gap detection is on by default. Projections can emit downstream events. Blue-green rebuild and async backfill ship for projections that grow into millions of events. End-to-end PII encryption applies across the event store, broker payloads, and structured logs through one shared conversion pipeline. All declarative through PHP attributes, on Laravel or Symfony.

This page walks the Ecotone capabilities in depth and closes with a head-to-head comparison against the PHP event-sourcing landscape.

## The Problem You Recognize

Your application stores current state. The customer disputes a charge, and the support team asks *"what exactly happened to this order?"* You read logs, joins, and updated-at timestamps, and you piece together a story.

A new read model is requested. You write a migration script — but the source data is "the current state of orders" and there's no way to verify the migration is correct because the events that produced the current state are gone.

A projection needs rebuilding because the rebuild logic changed. The single rebuild worker takes 14 hours on a 50-million-event stream. During those 14 hours, new events arrive and either block on the projection or risk being silently skipped because a concurrent transaction committed between two reads.

You want to add a new subscriber to the same event stream a year after the original projection shipped. The events are gone, replaced by a current-state table.

These are the symptoms Event Sourcing solves — *if* the event-sourcing implementation actually delivers a queryable history, parallel rebuild, gap detection, and projection-to-projection events.

## What the Industry Calls It

**Event Sourcing** — store every state change as an immutable event in an append-only stream; rebuild state by replaying events; treat the event log as the system of record and read models as derived views. The pattern compounds with CQRS (separate write and read models) and Domain-Driven Design (events as ubiquitous language).

In PHP, four serious implementations exist: Spatie laravel-event-sourcing, EventSauce, Patchlevel event-sourcing, and Ecotone. They diverge on the operationally-important parts: projection scaling, gap detection, projection emission, PII boundaries, and framework portability.

## How Ecotone Solves It

### Event-sourced aggregates as plain PHP

```php
#[EventSourcingAggregate]
final class Order
{
    use WithAggregateVersioning;

    #[Identifier] private string $orderId;
    private OrderStatus $status = OrderStatus::Placed;
    private int $amount = 0;

    #[CommandHandler]
    public static function place(PlaceOrder $command): array
    {
        return [new OrderWasPlaced($command->orderId, $command->amount)];
    }

    #[EventSourcingHandler]
    public function whenPlaced(OrderWasPlaced $event): void
    {
        $this->orderId = $event->orderId;
        $this->amount = $event->amount;
    }

    #[CommandHandler]
    public function pay(PayOrder $command): array
    {
        if ($this->status !== OrderStatus::Placed) {
            return [];
        }
        return [new OrderWasPaid($this->orderId, $command->paymentId)];
    }

    #[EventSourcingHandler]
    public function whenPaid(OrderWasPaid $event): void
    {
        $this->status = OrderStatus::Paid;
    }
}
```

Plain PHP class. Plain events. `#[CommandHandler]` records events; `#[EventSourcingHandler]` rebuilds state from the recorded history. Ecotone handles persistence, loading, and concurrency.

### Projections that scale, with gap detection and emission

```php
#[ProjectionV2(name: 'order_list')]
#[FromAggregateStream(Order::class)]
final class OrderListProjection
{
    #[EventHandler]
    public function whenPlaced(OrderWasPlaced $event, ProjectionState $state): void
    {
        // Build the read model. State is partitioned per Order::class, gap-aware,
        // and rebuildable in parallel from the event history.
    }
}
```

Three things to notice that PHP event-sourcing libraries usually don't deliver out of the box:

1. **Gap detection is on by default.** Events committed in concurrent transactions (where the gap-creator's transaction commits *after* the next event's commit) cannot be silently skipped. The projection tracks gaps and waits or fills them.
2. **Partitioned projections rebuild in parallel.** Use `#[Partitioned]` to declare per-aggregate partitioning, and the rebuild splits across N workers — 50 million events finish in roughly N times less wall-clock time, not 14 hours on one worker.
3. **Projections emit downstream events.** Use `EventStreamEmitter` inside a projection handler to publish a new event for sagas, other projections, or external handlers to subscribe via the normal `#[EventHandler]`. During rebuild, emission is automatically suppressed so downstream consumers aren't flooded with duplicate historical events.

### End-to-end PII encryption

```php
final class CustomerRegistered
{
    public function __construct(
        public readonly string $customerId,
        #[Sensitive] public readonly string $email,
        #[Sensitive] public readonly string $fullName,
    ) {}
}
```

One `#[Sensitive]` attribute encrypts the field in the event store, on the wire over RabbitMQ / SQS / Kafka / Redis / DBAL outbox, *and* in your structured logs — because all serialization flows through one shared conversion pipeline. Crypto-shred a customer by deleting their key; their events become unreadable everywhere they live.

### Blue-green rebuilds, async backfill, streaming projections

Ecotone Enterprise adds the operational tooling event-sourced production needs:
- **Blue-green deployments** — run a new projection version in parallel; switch traffic atomically once it catches up.
- **Async backfill** — push rebuild work to async workers; combine with partitioning to scale linearly with worker count.
- **Streaming projections** — consume events from Kafka or RabbitMQ Streams instead of the database event store, for cross-system integration and external event sources.

## How It Compares

| Dimension | Spatie laravel-event-sourcing | EventSauce | Patchlevel event-sourcing | Ecotone |
| --- | --- | --- | --- | --- |
| Framework support | Laravel only | Framework-agnostic | Framework-agnostic (Doctrine DBAL core, Symfony bundle ships) | Laravel + Symfony + Ecotone Lite |
| Projection rebuild | Single process, one cursor | Single process | Subscription engine, blue-green via subscriber-id, **single cursor per projection** | Per-projection cursors; partitioned rebuild scales across N workers |
| Gap detection | Not supported — concurrent-commit events can be silently skipped | Not supported | Opt-in | On by default with `GapAwarePosition` |
| Projection emission | Reactors handle live emission only — no auto-suppression on rebuild, so historical replays trigger duplicate downstream events | Not supported | Subscriptions can dispatch via bus, but no rebuild-time auto-suppression | `EventStreamEmitter` — emit downstream events from projections; auto-suppressed on rebuild |
| Per-handler isolation | Shared envelope; first throwing consumer affects siblings | Shared envelope; first throwing consumer aborts siblings; retry re-runs every consumer | Per-subscription isolation only — within a subscription, one failing handler advances no position; the subscription's whole pipeline blocks until resolved or skipped | Per-handler isolation — a copy of every message dispatched to every handler |
| PII encryption | Not supported | Not supported | Crypto-shredding inside the event store only | End-to-end: event store + broker payloads + structured logs through one pipeline |
| Multi-tenant projections | Manual | Manual | Manual | Dynamic table naming + tenant-routed channels |
| Streaming consumption (Kafka/RabbitMQ Streams) | Not supported | Not supported | Not supported | Streaming Projections |

Two framings worth holding together. **First, Spatie / EventSauce / Patchlevel are event-sourcing libraries** — they cover the ES slice. Everything around it (CQRS message buses, sagas as process managers, outbox, distributed bus, multi-tenant routing, async messaging, shared retry / DLQ / PII middleware) is your responsibility to assemble from other libraries. **Second, on the operational axes that matter at high event volume**, Ecotone covers parallel rebuild, default-on gap detection, projection emission with rebuild auto-suppression, end-to-end PII encryption, and Laravel + Symfony parity. Patchlevel is the closest PHP-native competitor on the ES axes — it has a subscription engine and per-subscription isolation that Spatie and EventSauce don't — but as a library it shares the same operational-burden shape, and the subscription engine's failure model (one failing handler blocks the subscription's whole pipeline) is the most consequential production gap.

## Next Steps

- [Event Sourcing Introduction](../modelling/event-sourcing/event-sourcing-introduction/) — event-sourced aggregates from scratch
- [Projections](../modelling/event-sourcing/setting-up-projections/) — read models, partitioning, gap detection
- [Emitting Events from Projections](../modelling/event-sourcing/setting-up-projections/emitting-events.md) — `EventStreamEmitter` and the rebuild-suppression rule
- [Gap Detection and Consistency](../modelling/event-sourcing/setting-up-projections/gap-detection-and-consistency.md) — how concurrent commits stay visible
- [Event Versioning](../modelling/event-sourcing/event-sourcing-introduction/event-versioning.md) — evolve events safely
- [PII Encryption (GDPR)](../modelling/event-sourcing/event-sourcing-introduction/persistence-strategy/event-serialization-and-pii-data-gdpr.md) — the `#[Sensitive]` attribute and the conversion pipeline
- [Audit Trail & State Rebuild](audit-trail-and-state-rebuild.md) — the audit-specific framing

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Partitioned Projections](../modelling/event-sourcing/setting-up-projections/scaling-and-advanced.md#partitioned-projections), [Async Backfill & Rebuild](../modelling/event-sourcing/setting-up-projections/backfill-and-rebuild.md), [Blue-Green Deployments](../modelling/event-sourcing/setting-up-projections/blue-green-deployments.md), and [Streaming Projections](../modelling/event-sourcing/setting-up-projections/scaling-and-advanced.md) — production tooling for event-sourced systems at scale.
{% endhint %}
