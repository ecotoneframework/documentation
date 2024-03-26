---
description: PHP Event Sourcing Persistence Strategy
---

# Persistence Strategy

## Persistence Strategy

Describes how streams with events will be stored.

### Single Stream Strategy

The default persistence strategy is the Single Stream Strategy.\
This persistence stores all instances of specific aggregate, within the same stream.

```php
#[ServiceContext]
public function persistenceStrategy(): EventSourcingConfiguration
{
    return \Ecotone\EventSourcing\EventSourcingConfiguration::createWithDefaults()
        ->withSingleStreamPersistenceStrategy();
}
```

```php
namespace Domain\Ticket;

#[EventSourcingAggregate]
class Ticket
```

All instances of `Ticket` will be stored within `Domain\Ticket\Ticket stream`.&#x20;

{% hint style="info" %}
Read more about this strategy under `SingleStreamStrategy`:\
[http://docs.getprooph.org/event-store/implementations/pdo\_event\_store/variants.html#SingleStreamStrategy](http://docs.getprooph.org/event-store/implementations/pdo\_event\_store/variants.html#SingleStreamStrategy)
{% endhint %}

### Stream Per Aggregate Strategy

This persistence creates a stream per aggregate instance.

```php
#[ServiceContext]
public function persistenceStrategy()
{
    return \Ecotone\EventSourcing\EventSourcingConfiguration::createWithDefaults()
        ->withStreamPerAggregatePersistenceStrategy();
}
```

```php
namespace Domain\Ticket;

#[EventSourcingAggregate]
class Ticket
```

Instances of `Ticket` will be stored within `Domain\Ticket\Ticket-{ticketId} stream` where `ticketId` is an identifier of a specific aggregate.&#x20;

{% hint style="info" %}
Read more about this strategy under `AggregateStreamStrategy`:\
[http://docs.getprooph.org/event-store/implementations/pdo\_event\_store/variants.html#AggregateStreamStrategy](http://docs.getprooph.org/event-store/implementations/pdo\_event\_store/variants.html#AggregateStreamStrategy)
{% endhint %}

### Custom Strategy

You may provide your own Customer Persistence Strategy as long as it implements `PersistenceStrategy`.

```php
#[ServiceContext]
public function aggregateStreamStrategy()
{
    return EventSourcingConfiguration::createWithDefaults()
        ->withCustomPersistenceStrategy(new CustomStreamStrategy(new FromProophMessageToArrayConverter()));
}
```

## Multiple Persistence Strategies

Once set, the persistence strategy will apply to all streams in your application. However, you may face a situation when you need to have a different strategy for one or more of your streams.

```php
#[ServiceContext]
public function eventSourcingConfiguration(): EventSourcingConfiguration
{
    return EventSourcingConfiguration::createWithDefaults()
        ->withSimpleStreamPersistenceStrategy()
        ->withPersistenceStrategyFor('some_stream', LazyProophEventStore::AGGREGATE_STREAM_PERSISTENCE)
    ;
}
```

The above will make the Simple Stream Strategy as default however, for `some_stream` Event Store will use the Aggregate Stream Strategy.

{% hint style="warning" %}
Please, be aware that you won't be able to set Custom Strategy that way.
{% endhint %}

## Custom Stream Name

If you want to make use of a custom stream name (the default is Aggregate class name), then you can apply `Stream` attribute to your aggregate.

```php
#[Stream("basket_stream")]
class Basket
```

Then tell the projection to make use of it:

```php
#[Projection(self::PROJECTION_NAME, "basket_stream")]
class BasketList
```

## Custom Aggregate Type

By default events in the stream will hold Aggregate Class name as `AggregateType`. \
You may customize this by applying `AggregateType` attribute to your Aggregate.

```php
#[AggregateType("basket")]
class Basket
```

{% hint style="success" %}
Setting this up will allow you to refactor the aggregate class name or namespace and still load the same Aggregate.
{% endhint %}
