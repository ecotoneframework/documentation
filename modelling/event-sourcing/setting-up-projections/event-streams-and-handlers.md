---
description: PHP Event Sourcing Projection Streams and Event Handlers
---

# Event Streams and Handlers

## The Problem

Your projection needs data from multiple aggregates — orders AND payments — or you want to handle only specific events instead of everything in the stream. How do you control what events reach your projection and how they are routed?

## Subscribing to Event Streams

Every Projection needs to declare which Event Streams it reads from. This tells Ecotone where to fetch events when the Projection is triggered.

### From Aggregate Stream (Recommended)

The most common case — subscribe to all events from a single aggregate type using `#[FromAggregateStream]`:

```php
#[ProjectionV2('ticket_list')]
#[FromAggregateStream(Ticket::class)]
class TicketListProjection
{
    #[EventHandler]
    public function onTicketRegistered(TicketWasRegistered $event): void
    {
        // handle event
    }
}
```

`#[FromAggregateStream(Ticket::class)]` automatically resolves both the stream name and the aggregate type from the `Ticket` class. This enables Ecotone to use the correct database indexes for fast event loading.

{% hint style="success" %}
Always prefer `#[FromAggregateStream]` when your aggregate class is available. It ensures optimal performance by providing the aggregate type metadata that enables indexed queries on the Event Store.
{% endhint %}

### From Multiple Aggregate Streams

When your Read Model combines data from multiple aggregates, use multiple `#[FromAggregateStream]` attributes:

```php
#[ProjectionV2('calendar_overview')]
#[FromAggregateStream(Calendar::class)]
#[FromAggregateStream(Meeting::class)]
class CalendarOverviewProjection
{
    #[EventHandler]
    public function onCalendarCreated(CalendarWasCreated $event): void
    {
        // handle calendar event
    }

    #[EventHandler]
    public function onMeetingScheduled(MeetingWasScheduled $event): void
    {
        // handle meeting event
    }
}
```

The Projection will process events from both streams, ordered by when they were stored.

### From a Named Stream

In some cases you may need to specify the stream name directly — for example when the aggregate class has been deleted or when targeting a custom stream name. Use `#[FromStream]` for this:

```php
#[ProjectionV2('legacy_tickets')]
#[FromStream(stream: 'ticket_stream', aggregateType: 'App\Domain\Ticket')]
class LegacyTicketProjection
{
    #[EventHandler]
    public function onTicketRegistered(TicketWasRegistered $event): void
    {
        // handle event from explicitly named stream
    }
}
```

{% hint style="warning" %}
When using `#[FromStream]`, always provide the `aggregateType` parameter. Without it, Ecotone cannot use the aggregate type index on the Event Store, resulting in significantly slower event loading — especially on large streams.
{% endhint %}

## Event Handler Routing

Ecotone routes events to the correct handler method. You have several options for controlling how this works.

### By Type Hint (Default)

The simplest approach — Ecotone routes based on the event class in the method signature:

```php
#[EventHandler]
public function onTicketRegistered(TicketWasRegistered $event): void
{
    // Only called for TicketWasRegistered events
}
```

### Named Events

If your events use `#[NamedEvent]` to decouple the stored event name from the PHP class name:

```php
#[NamedEvent('ticket.registered')]
class TicketWasRegistered
{
    public function __construct(
        public readonly string $ticketId,
        public readonly string $type
    ) {}
}
```

You can still type-hint your handler with the class — Ecotone automatically resolves the `#[NamedEvent]` mapping:

```php
#[EventHandler]
public function onTicketRegistered(TicketWasRegistered $event): void
{
    // Works automatically — Ecotone knows TicketWasRegistered maps to 'ticket.registered'
}
```

{% hint style="success" %}
You don't need to match the event name manually in `#[EventHandler('ticket.registered')]`. As long as the event class has `#[NamedEvent]`, type-hinting the class is enough — Ecotone handles the routing for you.
{% endhint %}

You can also subscribe by name explicitly, which is useful when you don't have (or don't want to import) the event class:

```php
#[EventHandler('ticket.registered')]
public function onTicketRegistered(array $event): void
{
    // Subscribe by name, receive raw array — no class dependency needed
}
```

### Catch-All Handler

To receive every event in the stream regardless of type:

```php
#[EventHandler('*')]
public function onAnyEvent(array $event): void
{
    // Called for every event in the stream
}
```

### Using Array Payload for Performance

When handling events by name, you can accept the raw array payload instead of a deserialized object. This skips deserialization and can significantly speed up processing — especially useful during [backfill or rebuild](backfill-and-rebuild.md) with large event volumes:

```php
#[EventHandler('ticket.registered')]
public function onTicketRegistered(array $event): void
{
    // $event is the raw array — no deserialization overhead
    $ticketId = $event['ticketId'];
}
```

{% hint style="success" %}
Using array payloads avoids the cost of deserializing event objects. When rebuilding a projection with thousands of events, this can make a noticeable difference in processing time.
{% endhint %}

## What's Next

Instead of writing raw SQL in your projections, you can use Ecotone's [Document Store](document-store-projection.md) for automatic serialization and storage — especially useful for rapid prototyping and simpler Read Models.
