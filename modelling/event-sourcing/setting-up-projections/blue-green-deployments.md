---
description: PHP Event Sourcing Projection Blue-Green Deployments
---

# Blue-Green Deployments

## The Problem

You need to add a column to your projection's table and change how events are processed. But rebuilding takes 30 minutes, and during that time your users see an empty dashboard. How do you deploy projection changes with zero downtime?

{% hint style="info" %}
The features described on this page are available as part of Ecotone Enterprise.
{% endhint %}

## The Blue-Green Strategy

Instead of rebuilding the existing projection in-place (which clears the data), deploy a **new version** alongside the old one:

1. **v1** continues serving traffic normally
2. **v2** is deployed and catches up from historical events in the background
3. Once v2 is fully caught up, switch traffic from v1 to v2
4. Delete v1

Both projections run against the **same Event Store** — no data migration or copying needed.

## Using #[ProjectionName] for Versioned Tables

The key mechanism is `#[ProjectionName]` — it injects the projection name as a parameter into your handlers. Use it to dynamically name your tables, so `tickets_v1` and `tickets_v2` coexist in the same database:

```php
#[ProjectionV2('tickets_v1')]
#[FromAggregateStream(Ticket::class)]
class TicketsProjection
{
    public function __construct(private Connection $connection) {}

    #[ProjectionInitialization]
    public function init(#[ProjectionName] string $projectionName): void
    {
        $this->connection->executeStatement(<<<SQL
            CREATE TABLE IF NOT EXISTS {$projectionName} (
                ticket_id VARCHAR(36) PRIMARY KEY,
                ticket_type VARCHAR(25),
                status VARCHAR(25)
            )
        SQL);
    }

    #[EventHandler]
    public function onTicketRegistered(
        TicketWasRegistered $event,
        #[ProjectionName] string $projectionName
    ): void {
        $this->connection->insert($projectionName, [
            'ticket_id' => $event->ticketId,
            'ticket_type' => $event->type,
            'status' => 'open',
        ]);
    }

    #[EventHandler]
    public function onTicketClosed(
        TicketWasClosed $event,
        #[ProjectionName] string $projectionName
    ): void {
        $this->connection->update(
            $projectionName,
            ['status' => 'closed'],
            ['ticket_id' => $event->ticketId]
        );
    }

    #[ProjectionDelete]
    public function delete(#[ProjectionName] string $projectionName): void
    {
        $this->connection->executeStatement("DROP TABLE IF EXISTS {$projectionName}");
    }

    #[ProjectionReset]
    public function reset(#[ProjectionName] string $projectionName): void
    {
        $this->connection->executeStatement("DELETE FROM {$projectionName}");
    }

    #[QueryHandler('getTickets')]
    public function getTickets(#[ProjectionName] string $projectionName): array
    {
        return $this->connection->fetchAllAssociative(
            "SELECT * FROM {$projectionName}"
        );
    }
}
```

Because the table name comes from the projection name, deploying `tickets_v2` creates a completely separate table — no conflicts with `tickets_v1`.

## Deploying Version 2

When you need to deploy changes, create v2 with `#[ProjectionDeployment]`:

```php
#[ProjectionV2('tickets_v2')]
#[FromAggregateStream(Ticket::class)]
#[ProjectionDeployment(manualKickOff: true, live: false)]
class TicketsV2Projection extends TicketsProjection
{
    // Same handlers — or modified handlers with your schema changes
    // The table name will be 'tickets_v2' thanks to #[ProjectionName]
}
```

Two settings control the deployment:

- **`manualKickOff: true`** — the projection won't auto-initialize. You control when it starts.
- **`live: false`** — events [emitted via EventStreamEmitter](emitting-events.md) are suppressed during the catch-up phase. This prevents duplicate notifications to downstream consumers.

## Step-by-Step Deployment Flow

### 1. Deploy v2

Deploy your code with the `tickets_v2` projection class. Because `manualKickOff: true`, nothing happens yet.

### 2. Initialize v2

Create the v2 table:

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:projection:init tickets_v2
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:init tickets_v2
```
{% endtab %}
{% endtabs %}

### 3. Backfill v2

Populate v2 with all historical events:

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:projection:backfill tickets_v2
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:backfill tickets_v2
```
{% endtab %}
{% endtabs %}

During this phase, v1 continues serving traffic normally. v2 processes historical events in the background.

### 4. Verify v2

Check that v2's data looks correct — query the `tickets_v2` table and compare with `tickets_v1`.

### 5. Switch Traffic

Update your application's query handlers to read from `tickets_v2` instead of `tickets_v1`.

### 6. Enable Live Mode

Update the v2 projection to remove `manualKickOff` and set `live: true`, so it processes new events and emits downstream events normally.

### 7. Delete v1

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:projection:delete tickets_v1
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:delete tickets_v1
```
{% endtab %}
{% endtabs %}

This calls `#[ProjectionDelete]` which drops the `tickets_v1` table.

```mermaid
sequenceDiagram
    participant App
    participant v1 as tickets_v1
    participant v2 as tickets_v2
    participant EventStore
    Note over App,v1: v1 serving traffic
    App->>v2: Deploy + Init
    EventStore->>v2: Backfill (historical events)
    Note over v2: Catching up...
    EventStore->>v1: Still processing live events
    EventStore->>v2: Caught up to present
    App->>v2: Switch queries to v2
    App->>v1: Delete v1
    Note over App,v2: v2 serving traffic, zero downtime
```

## Event Emission Control

The `live: false` setting is critical for projections that [emit events](emitting-events.md). Without it, the backfill phase would re-emit all historical events — sending thousands of duplicate notifications to downstream consumers.

With `live: false`:
- Events emitted during backfill are silently discarded
- Once you switch to `live: true`, new events are emitted normally
- Downstream consumers only see events once

## Upgrading from Global to Partitioned

Blue-green deployments also work for changing projection types. You can deploy v2 as a [Partitioned Projection](scaling-and-advanced.md) alongside your existing global v1:

```php
#[ProjectionV2('tickets_v2')]
#[FromAggregateStream(Ticket::class)]
#[Partitioned]
#[ProjectionDeployment(manualKickOff: true, live: false)]
class TicketsV2Projection extends TicketsProjection
{
    // Same handlers, now partitioned
}
```

The same Event Store backs both projections. The only difference is how events are tracked and processed — v2 uses per-aggregate partitions instead of a single global position. Once v2 catches up, switch traffic and delete v1.
