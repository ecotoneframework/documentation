---
description: Upgrading from Projection V1 to ProjectionV2
---

# Upgrading from V1 to V2

## The Problem

You have existing projections using the old `#[Projection]` API and want to migrate to `#[ProjectionV2]`. Do you need to migrate data? Will there be downtime?

## No Data Migration Needed

Both V1 and V2 projections read from the **same underlying Event Store**. The events don't change — only the projection infrastructure does. This means upgrading is purely about registering a new projection, not migrating data.

## Upgrade Steps

### 1. Create the V2 Projection

Take your existing V1 projection and create a V2 version alongside it. The main changes are:

- Replace `#[Projection("name", Aggregate::class)]` with `#[ProjectionV2('name_v2')]` + `#[FromAggregateStream(Aggregate::class)]`
- Keep your event handlers and lifecycle hooks as they are

**V1 (existing):**

```php
#[Projection('ticket_list', Ticket::class)]
class TicketListProjection
{
    #[EventHandler]
    public function onTicketRegistered(TicketWasRegistered $event): void
    {
        $this->connection->insert('ticket_list', [
            'ticket_id' => $event->ticketId,
            'ticket_type' => $event->type,
            'status' => 'open',
        ]);
    }

    #[ProjectionInitialization]
    public function init(): void { /* CREATE TABLE ticket_list */ }
}
```

**V2 (new):**

```php
#[ProjectionV2('ticket_list_v2')]
#[FromAggregateStream(Ticket::class)]
class TicketListV2Projection
{
    #[EventHandler]
    public function onTicketRegistered(TicketWasRegistered $event): void
    {
        $this->connection->insert('ticket_list_v2', [
            'ticket_id' => $event->ticketId,
            'ticket_type' => $event->type,
            'status' => 'open',
        ]);
    }

    #[ProjectionInitialization]
    public function init(): void { /* CREATE TABLE ticket_list_v2 */ }

    #[ProjectionDelete]
    public function delete(): void { /* DROP TABLE ticket_list_v2 */ }

    #[ProjectionReset]
    public function reset(): void { /* DELETE FROM ticket_list_v2 */ }
}
```

{% hint style="success" %}
Both projections can run side by side — they read from the same Event Store but write to different tables. There is no conflict.
{% endhint %}

### 2. Initialize and Backfill

Deploy the V2 projection, then initialize and backfill it:

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:projection:init ticket_list_v2
bin/console ecotone:projection:backfill ticket_list_v2
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:init ticket_list_v2
artisan ecotone:projection:backfill ticket_list_v2
```
{% endtab %}
{% endtabs %}

The V2 projection will process all historical events from the Event Store and catch up to the present.

### 3. Verify

Compare the V2 read model against V1 to confirm the data matches:

```php
$v1Tickets = $connection->fetchAllAssociative('SELECT * FROM ticket_list ORDER BY ticket_id');
$v2Tickets = $connection->fetchAllAssociative('SELECT * FROM ticket_list_v2 ORDER BY ticket_id');

assert($v1Tickets === $v2Tickets, 'V1 and V2 data should match');
```

### 4. Switch Traffic

Once verified, update your application's query handlers to read from the V2 table. Then remove the V1 projection.

## What Changes Between V1 and V2

| Aspect | V1 (`#[Projection]`) | V2 (`#[ProjectionV2]`) |
|--------|------|------|
| **Stream declaration** | `#[Projection("name", Aggregate::class)]` | `#[ProjectionV2('name')]` + `#[FromAggregateStream(Aggregate::class)]` |
| **Position tracking** | Prooph-based, stored in projections table | Ecotone-native, stored in projection state table |
| **Gap detection** | Time-based (blocking) | [Track-based (non-blocking)](gap-detection-and-consistency.md) |
| **Partitioning** | Not available | `#[Partitioned]` |
| **Backfill** | Manual reset + trigger | `ecotone:projection:backfill` CLI command |
| **Rebuild** | Manual reset + trigger | `ecotone:projection:rebuild` CLI command |
| **Blue-green** | Not available | `#[ProjectionDeployment]` |
| **Flush mechanism** | Per-event persistence | [Configurable batch commits](execution-modes.md#batch-size-and-flushing) |

{% hint style="info" %}
V1 projections continue to work. You can migrate at your own pace — there is no deadline to switch.
{% endhint %}
