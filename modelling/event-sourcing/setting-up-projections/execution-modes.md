---
description: PHP Event Sourcing Projection Execution Modes
---

# Execution Modes

## The Problem

Your projection runs in the same request as the command handler, and under heavy load it slows down your API. Or you have multiple projections and don't want one slow projection to block others. How do you control when and where projections execute, and what consistency trade-offs come with each choice?

Execution modes determine **where** your projection runs (same process or background worker) and **when** it processes events (immediately or later). Each mode comes with different consistency guarantees.

## Choosing the Right Mode

This is about **where** and **when** execution happens, and what **consistency consequences** you accept:

| Feature | Sync Event-Driven | Async Event-Driven | Polling (Enterprise) |
|---------|:-:|:-:|:-:|
| Consistency | Immediate | Eventual | Eventual |
| Transaction | Same as command | Batched, per-batch commits | Batched, per-batch commits |
| Triggering | On event publish | On event via channel | Polls database at intervals |
| Best for | Low write volume, testing | Production workloads | Dedicated background worker |

{% hint style="info" %}
Execution mode does not affect horizontal scaling. For parallel processing across multiple aggregates, see [Scaling and Advanced](scaling-and-advanced.md) — which uses Partitioned or Streaming projections (Enterprise).
{% endhint %}

{% hint style="success" %}
You can start with synchronous projections for simplicity, and switch to asynchronous later by adding a single attribute — no code changes needed in your projection handlers.
{% endhint %}


## Synchronous Event-Driven (Default)

By default, projections execute synchronously — in the same process and the same database transaction as the Command Handler that produced the events.

```php
#[ProjectionV2('ticket_list')]
#[FromAggregateStream(Ticket::class)]
class TicketListProjection
{
    // No additional attributes needed — synchronous is the default
    
    #[EventHandler]
    public function onTicketRegistered(TicketWasRegistered $event): void
    {
        // This runs in the same transaction as the command
    }
}
```

{% hint style="success" %}
Synchronous projections run within the same database transaction as the Event Store changes. When you query the Read Model right after a command, you always get consistent, up-to-date data.
{% endhint %}

**When to use:**
- Low write volume — a few writes per second
- Testing — immediate feedback, no async complexity
- Simple applications — where eventual consistency adds unnecessary complexity

**Trade-off:** If the projection is slow (complex queries, external calls), it slows down the entire command handling. For high-throughput scenarios, consider asynchronous execution.

## Asynchronous Event-Driven

To decouple the projection from the command handler, mark it as asynchronous. The event is delivered via a message channel and processed by a background worker:

```php
#[Asynchronous('projections')]
#[ProjectionV2('ticket_list')]
#[FromAggregateStream(Ticket::class)]
class TicketListProjection
{
    #[EventHandler]
    public function onTicketRegistered(TicketWasRegistered $event): void
    {
        // This runs in a separate process, triggered by the message channel
    }
}
```

The projection code stays exactly the same — you just add `#[Asynchronous('projections')]`. Ecotone handles delivering the trigger event via the `projections` channel.

To start the background worker:

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:run projections -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:run projections -vvv
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->run('projections');
```
{% endtab %}
{% endtabs %}

**When to use:**
- High write volume — projection processing shouldn't slow down commands
- Multiple projections — each can process at its own pace
- Production workloads — decoupled, resilient processing

{% hint style="info" %}
Multiple projections can share the same async channel (same consumer process), or each can have its own dedicated channel.
{% endhint %}

**Trade-off:** Data in the Read Model may be slightly behind the Event Store (eventual consistency). If you query immediately after a command, you might get stale results.

## Batch Size and Flushing

By default, projections load up to 1000 events per batch. You can customize this with `#[ProjectionExecution]`:

```php
#[ProjectionV2('ticket_list')]
#[FromAggregateStream(Ticket::class)]
#[ProjectionExecution(eventLoadingBatchSize: 500)]
class TicketListProjection
{
    // Events are loaded and processed 500 at a time
}
```

### How Batching Works

Events are processed in batches, and each batch is wrapped in its own database transaction. After each batch:

1. `#[ProjectionFlush]` handler is called (if defined)
2. The projection's position is saved
3. The transaction is committed

This prevents one massive transaction from locking your database tables for the entire projection run. Even if you have 100,000 events to process, the database is only locked for one batch at a time.

```php
#[ProjectionFlush]
public function flush(): void
{
    // Called after each batch of events is processed
    // Useful for flushing buffers, clearing caches, etc.
}
```

{% hint style="success" %}
Ecotone automatically manages transactions at batch boundaries. In async mode, each batch gets its own transaction — not the entire message processing. If you use Doctrine ORM, Ecotone also flushes and clears the EntityManager at batch boundaries automatically, preventing memory leaks.
{% endhint %}

## Polling (Enterprise)

Polling projections run as a dedicated background process that periodically queries the event store for new events:

```php
#[ProjectionV2('ticket_list')]
#[FromAggregateStream(Ticket::class)]
#[Polling('ticket_list_poller')]
class TicketListProjection
{
    #[EventHandler]
    public function onTicketRegistered(TicketWasRegistered $event): void
    {
        // Executed when the poller finds new events
    }
}
```

{% hint style="info" %}
Polling projections are available as part of Ecotone Enterprise.
{% endhint %}