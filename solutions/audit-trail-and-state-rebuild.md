---
description: How to implement Event Sourcing for audit trails and state rebuilds in PHP
---

# Audit Trail & State Rebuild

## The Problem You Recognize

A customer disputes a charge. Your support team asks "what exactly happened to this order?" The answer requires reading application logs, database timestamps, and hoping someone didn't overwrite the data.

Your read models need a schema change. You write a migration script, but there's no way to verify the migrated data is correct — the original events that created it are gone. You store the current state, but not how you got there.

The symptoms:
- **No history** — you know what the current price is, but not what it was yesterday
- **Risky migrations** — changing the read model means writing one-off scripts and praying
- **Compliance gaps** — auditors ask for a complete trail of changes and you can't provide one

## What the Industry Calls It

**Event Sourcing** — instead of storing the current state, store the sequence of events that led to it. Rebuild any view of the data by replaying events. Get a complete, immutable audit trail for free.

## How Ecotone Solves It

Ecotone provides Event Sourcing as a first-class feature with built-in projections. Your aggregate records events instead of mutating state:

```php
#[EventSourcingAggregate]
class Order
{
    #[Identifier]
    private string $orderId;

    #[CommandHandler]
    public static function place(PlaceOrder $command): array
    {
        return [new OrderWasPlaced($command->orderId, $command->items)];
    }

    #[EventSourcingHandler]
    public function onOrderPlaced(OrderWasPlaced $event): void
    {
        $this->orderId = $event->orderId;
    }
}
```

Build read models (projections) that can be rebuilt at any time from the event history:

```php
#[ProjectionV2('order_list')]
#[FromAggregateStream(Order::class)]
class OrderListProjection
{
    #[EventHandler]
    public function onOrderPlaced(OrderWasPlaced $event): void
    {
        // Build your read model — rebuildable from history
    }
}
```

Works with **Postgres, MySQL, and MariaDB** for event storage. Projections can write to any storage you choose.

## Next Steps

- [Event Sourcing Introduction](../modelling/event-sourcing/event-sourcing-introduction/) — How Event Sourced Aggregates work
- [Projections](../modelling/event-sourcing/setting-up-projections/) — Build and rebuild read models
- [Event Versioning](../modelling/event-sourcing/event-sourcing-introduction/event-versioning.md) — Evolve your events safely
- [Event Stream Persistence](../modelling/event-sourcing/event-sourcing-introduction/persistence-strategy.md) — Storage strategies and snapshots

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Partitioned Projections](../modelling/event-sourcing/setting-up-projections/scaling-and-advanced.md#partitioned-projections) for independent per-aggregate processing, [Async Backfill & Rebuild](../modelling/event-sourcing/setting-up-projections/backfill-and-rebuild.md) with parallel workers, and [Blue-Green Deployments](../modelling/event-sourcing/setting-up-projections/blue-green-deployments.md) for zero-downtime projection updates.
{% endhint %}
