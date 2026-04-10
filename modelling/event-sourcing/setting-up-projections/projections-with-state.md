---
description: PHP Event Sourcing Stateful Projections
---

# Projections with State

## The Problem

You need to count all tickets or calculate a running total across events, but you don't want to create a database table just for a counter. How do you keep state between event handler calls without external storage?

## Projection State

Ecotone allows projections to carry **internal state** that is automatically persisted between executions. The state is passed to each event handler and can be updated by returning a new value.

This is useful for:
- Counters and aggregates (total count, running average)
- Throw-away projections that calculate a result, [emit an event](emitting-events.md), and then get [deleted](lifecycle-management.md#delete-a-projection)
- Projections that don't need an external database table

## Passing State Inside Projection

Mark a method parameter with `#[ProjectionState]` to receive the current state. Return the updated state from the handler:

```php
#[ProjectionV2('ticket_counter')]
#[FromAggregateStream(Ticket::class)]
class TicketCounterProjection
{
    #[EventHandler]
    public function when(
        TicketWasRegistered $event,
        #[ProjectionState] TicketCounterState $state
    ): TicketCounterState {
        return $state->increase();
    }
}
```

Ecotone resolves the `#[ProjectionState]` parameter and passes the current state. The **returned value** becomes the new state for the next event handler call.

State is shared across all event handlers in the same projection — if handler A updates the state, handler B receives the updated version.

{% hint style="success" %}
The state can be a simple array or a class. Ecotone automatically serializes and deserializes it for you.
{% endhint %}

## Fetching the State from Outside

To read projection state from other parts of your application, create a Gateway interface with `#[ProjectionStateGateway]`.

### Global Projection State

For a globally tracked projection, the gateway has no parameters — there's only one state to fetch:

```php
interface TicketCounterGateway
{
    #[ProjectionStateGateway(TicketCounterProjection::NAME)]
    public function getCounter(): TicketCounterState;
}
```

Ecotone automatically converts the stored state (array or serialized data) to the declared return type (`TicketCounterState`). If you have a converter registered, it will be used:

```php
#[Converter]
public function toCounterState(array $state): CounterState
{
    return new CounterState(
        ticketCount: $state['ticketCount'] ?? 0,
        closedTicketCount: $state['closedTicketCount'] ?? 0,
    );
}
```

{% hint style="success" %}
Gateways are automatically registered in your Dependency Container, so you can inject them like any other service.
{% endhint %}

### Partitioned Projection State (Enterprise)

For a partitioned projection, each aggregate has its own state. Pass the aggregate ID as the first parameter:

```php
interface TicketCounterGateway
{
    #[ProjectionStateGateway('ticket_counter')]
    public function fetchStateForPartition(string $aggregateId): CounterState;
}
```

Usage:

```php
$gateway = $container->get(TicketCounterGateway::class);

// Fetch state for a specific aggregate
$stateForTicket1 = $gateway->fetchStateForPartition('ticket-1');
$stateForTicket2 = $gateway->fetchStateForPartition('ticket-2');
```

Ecotone resolves the stream name and aggregate type from the projection's configuration, then composes the partition key internally. You only need to pass the aggregate ID — the rest is handled automatically.

### Multi-Stream Partitioned Projections (Enterprise)

When a partitioned projection reads from **multiple streams**, Ecotone needs to know which stream the aggregate ID belongs to. Use `#[FromAggregateStream]` on the gateway method to disambiguate:

```php
interface CalendarCounterGateway
{
    #[ProjectionStateGateway('calendar_counter')]
    #[FromAggregateStream(Calendar::class)]
    public function fetchCalendarState(string $aggregateId): CounterState;

    #[ProjectionStateGateway('calendar_counter')]
    #[FromAggregateStream(Meeting::class)]
    public function fetchMeetingState(string $aggregateId): CounterState;
}
```

Each method targets a specific stream — so you can fetch state for a Calendar aggregate or a Meeting aggregate from the same multi-stream projection.

{% hint style="info" %}
`#[FromAggregateStream]` on the gateway method is only needed when the projection reads from multiple streams. For single-stream projections, Ecotone resolves the stream automatically.
{% endhint %}

## High-Performance Projections with Flush State (Enterprise)

For projections that need to process large volumes of events quickly — during backfill or rebuild — you can combine `#[ProjectionState]` with `#[ProjectionFlush]` to build extremely performant projections.

The idea: instead of doing a database INSERT on every single event, you **accumulate state in memory** across the entire batch, and then persist it in one operation during flush.

```php
#[ProjectionV2('ticket_stats')]
#[FromAggregateStream(Ticket::class)]
#[ProjectionExecution(eventLoadingBatchSize: 1000)]
class TicketStatsProjection
{
    public function __construct(private Connection $connection) {}

    #[EventHandler]
    public function onTicketRegistered(
        TicketWasRegistered $event,
        #[ProjectionState] array $state
    ): array {
        // No database call — just update in-memory state
        $type = $event->type;
        $state[$type] = ($state[$type] ?? 0) + 1;
        return $state;
    }

    #[EventHandler]
    public function onTicketClosed(
        TicketWasClosed $event,
        #[ProjectionState] array $state
    ): array {
        $state['closed'] = ($state['closed'] ?? 0) + 1;
        return $state;
    }

    #[ProjectionFlush]
    public function flush(#[ProjectionState] array $state): void
    {
        // One database operation per batch instead of per event
        foreach ($state as $type => $count) {
            $this->connection->executeStatement(
                'INSERT INTO ticket_stats (type, count) VALUES (?, ?) 
                 ON DUPLICATE KEY UPDATE count = ?',
                [$type, $count, $count]
            );
        }
    }

    #[ProjectionInitialization]
    public function init(): void
    {
        $this->connection->executeStatement(<<<SQL
            CREATE TABLE IF NOT EXISTS ticket_stats (
                type VARCHAR(50) PRIMARY KEY,
                count INT NOT NULL DEFAULT 0
            )
        SQL);
    }
}
```

With a batch size of 1000, this projection processes 1000 events without a single database write, then does one bulk persist during flush. During a rebuild over millions of events, this is dramatically faster than writing on every event.

{% hint style="info" %}
Using `#[ProjectionState]` in `#[ProjectionFlush]` methods is available as part of Ecotone Enterprise.
{% endhint %}

{% hint style="success" %}
Ecotone takes care of persisting and loading the state between batches automatically. You only need to focus on the accumulation logic in event handlers and the persistence logic in flush. This pattern is ideal for projections that need to rebuild quickly over large event streams.
{% endhint %}

## Demo

[Example implementation using Ecotone Lite.](https://github.com/ecotoneframework/quickstart-examples/tree/master/StatefulProjection)
