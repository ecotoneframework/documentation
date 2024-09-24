---
description: PHP Event Streams
---

# Choosing Event Streams for Projection

## Choosing Event Streams for Projection

The `Projection` is deriving from `Event Stream.`\
There may be situations when we will want to derive the projection from multiple streams however. \
Let's see what options do we have:

### From Single Stream

If we are interested in single stream, we can listen directly for specific aggregate

```php
#[Projection("basketList", Basket::class)]
class BasketList
{
    #[EventHandler]
    public function addBasket(BasketWasCreated $event) : void
    {
        // do something
    }
}
```

In here we are handling events from single `Basket's Aggregate stream`. It will contain all the events in relation to this aggregate.

### From Multiple Streams

There may be situations, when we will want to handle different streams together.

```php
#[Projection("log_projection", [Ticket::class, Basket::class])]
class Logger
```

### From Category

In case if using [`Stream Per Aggregate Persistence Strategy`](../event-sourcing-introduction/persistence-strategy/#stream-per-aggregate-strategy) we will need to use categories to target.\
If we would listen on `Domain\Ticket` stream using `Stream Per Aggregate` then we would not target any event, as the streams that are created are suffixed by the identifier `Domain\Ticket-123`.&#x20;

In that case we can make use of categories in order to target `Domain\Ticket-*.`

```php
#[Projection("category_projection", fromCategories: Ticket::class)]
class FromCategoryUsingAggregatePerStreamProjection
```

## Handling Events By Names

If you want to avoid storing class names of your events in the `Event Store` you may mark them with name.

```php
#[NamedEvent("basket.was_created")]
class BasketWasCreated
{
    public const EVENT_NAME = "basket.was_created";

    private string $id;

    public function __construct(string $id)
    {
        $this->id = $id;
    }

    public function getId(): string
    {
        return $this->id;
    }
}
```

And tell the projection to make use of it

```php
#[Projection("basket_list")]
class BasketList
{
    #[EventHandler("basket.was_created")]
    public function addBasket(BasketWasCreated $event) : void
    {
        // do something with $event
    }
```

### Speeding Up Projection Restore

If projections are handling the events by names, then there is no need to deserialization of the event to the class and simple array can be used. In case of thousands of events during [resetting the projection](../#resetting-the-projection) it will speed up the process.

```php
    #[EventHandler("basket.was_created")]
    public function addBasket(array $event) : void
    {
        // do something with $event
    }
```
