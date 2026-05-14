---
description: PHP Event Sourcing Projection Backfill and Rebuild
---

# Backfill and Rebuild

## The Problem

You deployed a new "order analytics" projection to production, but it only processes events from now on. You have 2 years of order history sitting in the event store. How do you populate the projection with historical data? And later, when you fix a bug in the projection logic, how do you replay everything?

## Choosing the Right Strategy

Before reaching for rebuild, consider the lighter alternatives. The cheapest fix is the one that doesn't require replaying any history at all.

| Situation | Strategy | Cost |
|---|---|---|
| Adding a column where historical rows can use a default value | **Default value migration** (no replay) | Five-minute deploy |
| Adding a brand-new projection that has no data yet | **Backfill** | One pass over history |
| Fixing a bug in a small projection where downtime during rebuild is acceptable | **Rebuild** (in-place) | Read model empty during rebuild |
| Fixing a bug in a large or user-facing projection | **[Blue-green deployment](blue-green-deployments.md)** | v1 keeps serving while v2 catches up |

### The No-Rebuild Tactic: Default Values

If you are adding a new column to a projection, the first conversation to have is with Product, not with ops: **can historical rows use a default value?**

Often yes. "We're adding a `priority` column — historical tickets without explicit priority can show as `normal`" is a five-minute deploy: extend `#[ProjectionInitialization]` with an idempotent migration, and the new handler computes the real value for events from now on.

```php
#[ProjectionInitialization]
public function init(): void
{
    $this->connection->executeStatement(<<<SQL
        CREATE TABLE IF NOT EXISTS ticket_list (
            ticket_id VARCHAR(36) PRIMARY KEY,
            ticket_type VARCHAR(25),
            status VARCHAR(25)
        )
    SQL);

    $this->connection->executeStatement(<<<SQL
        ALTER TABLE ticket_list
        ADD COLUMN IF NOT EXISTS priority VARCHAR(25) NOT NULL DEFAULT 'normal'
    SQL);
}
```

`#[ProjectionInitialization]` re-runs on every deploy, and `IF NOT EXISTS` keeps both statements idempotent. Historical rows get `'normal'`; new tickets get their real priority from the updated handler. No rebuild, no backfill, no downtime.

This will not always work — sometimes you genuinely need to recompute historical rows. But when it does, it skips the entire backfill/rebuild discussion.

## Backfill — Populating a New Projection

**Backfill** processes all historical events from position 0 to the current position. It's used when you deploy a fresh projection and need to populate it with past data.

### Sync Backfill

Add `#[ProjectionBackfill]` to your projection and run the CLI command:

```php
#[ProjectionV2('order_analytics')]
#[FromAggregateStream(Order::class)]
#[ProjectionBackfill]
class OrderAnalyticsProjection
{
    #[EventHandler]
    public function onOrderPlaced(OrderWasPlaced $event): void
    {
        // This will process ALL historical OrderWasPlaced events during backfill
    }

    #[ProjectionInitialization]
    public function init(): void { /* CREATE TABLE */ }

    #[ProjectionReset]
    public function reset(): void { /* DELETE FROM */ }
}
```

Then run:

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:projection:backfill order_analytics
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:backfill order_analytics
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand('ecotone:projection:backfill', ['name' => 'order_analytics']);
```
{% endtab %}
{% endtabs %}

The backfill reads all events from the beginning of the stream, processing them in [configurable batches](execution-modes.md#batch-size-and-flushing). After backfill completes, the projection is caught up and will process new events as they arrive.

{% hint style="success" %}
Backfill runs synchronously and is available in the open-source edition.
{% endhint %}

### Async Backfill (Enterprise)

For large event stores with millions of events, synchronous backfill may take too long — it runs in the CLI process and blocks until all events are processed. By setting `asyncChannelName`, the backfill command instead **dispatches messages** to a channel, turning the backfill into an asynchronous background process:

```php
#[ProjectionV2('order_analytics')]
#[FromAggregateStream(Order::class)]
#[ProjectionBackfill(asyncChannelName: 'backfill_channel')]
class OrderAnalyticsProjection
{
    // Same handlers as above
}
```

Run the backfill command (dispatches messages instantly), then start workers to process them:

{% tabs %}
{% tab title="Symfony" %}
```bash
# Dispatches backfill messages to the channel
bin/console ecotone:projection:backfill order_analytics

# Start the worker (only ONE for a global projection — see note below)
bin/console ecotone:run backfill_channel -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:backfill order_analytics
artisan ecotone:run backfill_channel -vvv
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
**Run only one worker for a global projection.** The position tracker is a single value: two workers consuming the same backfill channel would race to advance it, and one would overwrite the other's commit. Multiple workers only help once you can describe each unit of work as "one aggregate's events" — which is what partitioning gives you (next section).
{% endhint %}

### Scaling Async Backfill with Partitioned Projections

The real power of async backfill comes when combined with `#[Partitioned]`. Each partition (aggregate) can be backfilled independently, so the work is split into batches that multiple workers process in parallel:

```php
#[ProjectionV2('order_analytics')]
#[FromAggregateStream(Order::class)]
#[Partitioned]
#[ProjectionBackfill(backfillPartitionBatchSize: 100, asyncChannelName: 'backfill_channel')]
class OrderAnalyticsProjection
{
    // Same handlers
}
```

When you run the backfill command with 10,000 aggregates and `backfillPartitionBatchSize: 100`:

1. Ecotone dispatches **100 messages** to `backfill_channel` (10,000 / 100)
2. Each message backfills 100 partitions
3. Start 4 workers → 4 batches processed in parallel → **4x faster**
4. Start 10 workers → **10x faster**

{% hint style="success" %}
With partitioned projections, both backfill and rebuild scale linearly with worker count. A backfill that takes 2 hours with 1 worker takes 12 minutes with 10 workers.
{% endhint %}

{% hint style="info" %}
Async backfill is available as part of Ecotone Enterprise.
{% endhint %}

## Rebuild — Reset and Replay (Enterprise)

**Rebuild** is different from backfill: it **resets** an existing projection (clears data and position) and then **replays** all events from the beginning.

Use rebuild when:
- You fixed a bug in a handler and the Read Model has incorrect data
- You changed the projection's schema and need to reprocess everything
- You want to add a new event handler to an existing projection and apply it retroactively

{% hint style="info" %}
Rebuild is available as part of Ecotone Enterprise.
{% endhint %}

How rebuild works depends on the projection type — and the difference is significant.

### Rebuilding a Global Projection

For a globally tracked projection, rebuild works as **reset + backfill** on the entire dataset:

1. `#[ProjectionReset]` is called — clears **all** data (e.g., `DELETE FROM ticket_list`)
2. Position is reset to the beginning
3. All events in the stream are replayed through the handlers

```php
#[ProjectionV2('ticket_list')]
#[FromAggregateStream(Ticket::class)]
#[ProjectionRebuild]
class TicketListProjection
{
    #[ProjectionReset]
    public function reset(): void
    {
        // Clears ALL data — entire table
        $this->connection->executeStatement('DELETE FROM ticket_list');
    }

    // ... event handlers
}
```

{% hint style="warning" %}
Global rebuild deletes all data first, then repopulates. During the rebuild window, the Read Model is empty or incomplete. This can also lock the table depending on your database. For zero-downtime alternatives, see [Blue-Green Deployments](blue-green-deployments.md).
{% endhint %}

### Rebuilding a Partitioned Projection

For partitioned projections, rebuild is **much safer**. Instead of resetting the entire projection at once, Ecotone rebuilds **each partition (aggregate) separately**:

1. For each partition: within a transaction, delete that partition's projected data and re-project it
2. Other partitions are unaffected — they continue serving reads normally
3. Only one aggregate's data is unavailable at a time, and only briefly

```php
#[ProjectionV2('ticket_details')]
#[FromAggregateStream(Ticket::class)]
#[Partitioned]
#[ProjectionRebuild(partitionBatchSize: 50)]
class TicketDetailsProjection
{
    #[ProjectionReset]
    public function reset(#[PartitionAggregateId] string $aggregateId): void
    {
        // Resets only THIS aggregate's data — not the whole table
        $this->connection->executeStatement(
            'DELETE FROM ticket_details WHERE ticket_id = ?',
            [$aggregateId]
        );
    }

    // ... event handlers
}
```

Notice the key difference: `#[ProjectionReset]` receives `#[PartitionAggregateId]` — it only deletes the data for the specific aggregate being rebuilt, not the entire table.

{% hint style="success" %}
**Partitioned rebuilds also isolate failures.** If a handler bug only triggers on one aggregate's specific event sequence, only *that* partition gets stuck. The rest of the rebuild keeps progressing across the other aggregates. You can investigate the failing partition without an entire rebuild stalling for hours waiting for someone to wake up — and once you ship the fix, the stuck partition retries from where it stopped.

A global rebuild has the opposite property: one bad partition blocks the whole queue.
{% endhint %}

### Controlling Rebuild Batch Size

The `partitionBatchSize` parameter controls how many partitions are processed per rebuild command:

```php
#[ProjectionRebuild(partitionBatchSize: 50)]
```

With 1000 aggregates and `partitionBatchSize: 50`, Ecotone dispatches 20 rebuild commands — each processing 50 partitions.

### Scaling Rebuild with Async Workers

For large projections, you can distribute rebuild work across multiple workers:

```php
#[ProjectionV2('ticket_details')]
#[FromAggregateStream(Ticket::class)]
#[Partitioned]
#[ProjectionRebuild(partitionBatchSize: 50, asyncChannelName: 'rebuild_channel')]
class TicketDetailsProjection
{
    // ... same as above
}
```

When you run `ecotone:projection:rebuild ticket_details`:

1. Ecotone counts the partitions (e.g., 1000 aggregates)
2. Divides them into batches of 50 → 20 messages
3. Sends all 20 messages to `rebuild_channel`
4. Multiple workers consume from `rebuild_channel` in parallel
5. Each worker rebuilds its batch of 50 partitions independently

This means you can rebuild a projection with millions of aggregates by simply scaling up your worker count. Just like with [async backfill](#scaling-async-backfill-with-partitioned-projections), throughput scales linearly with the number of workers.

Run the rebuild command, then start workers:

{% tabs %}
{% tab title="Symfony" %}
```bash
# Trigger the rebuild (dispatches messages)
bin/console ecotone:projection:rebuild ticket_details

# Start workers (run multiple for parallel processing)
bin/console ecotone:run rebuild_channel -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:rebuild ticket_details
artisan ecotone:run rebuild_channel -vvv
```
{% endtab %}
{% endtabs %}

### Sync Rebuild

Without `asyncChannelName`, rebuild runs synchronously — all partitions are processed in the current process:

```php
#[ProjectionRebuild(partitionBatchSize: 50)]
// No asyncChannelName — rebuild happens immediately during CLI command
```

{% hint style="warning" %}
During rebuild, the Read Model is being repopulated. If you need zero-downtime rebuilds, see [Blue-Green Deployments](blue-green-deployments.md).
{% endhint %}

## Backfill vs Rebuild

| | Backfill | Rebuild (Global) | Rebuild (Partitioned) |
|---|---|---|---|
| **Purpose** | Populate a new, empty projection | Fix existing projection | Fix existing projection |
| **Starting state** | Fresh (no data) | All data cleared first | Per-partition data cleared |
| **Calls reset?** | No | Yes — entire table | Yes — per aggregate |
| **Impact during run** | None (table is new) | Table empty until done | Only one aggregate briefly affected |
| **Parallel workers?** | Via async backfill | Via async channel | Via async channel + partition batches |
| **When to use** | First deployment | Bug fix (simple projections) | Bug fix (production, at scale) |
| **Open source?** | Yes (sync) | Enterprise | Enterprise |
