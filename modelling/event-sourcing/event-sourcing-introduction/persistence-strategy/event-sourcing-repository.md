# Event Sourcing Repository

Ecotone comes with inbuilt Event Sourcing repository after [Event Sourcing package is installed](../../installation.md). However you want to roll out your own storage for Events, or maybe you already use some event-sourcing framework and would like to integrate with it. \
For this you can take over the control by introducing your own Event Sourcing Repository.

{% hint style="warning" %}
Using Custom Event Sourcing Repository will not allow you to make use of [inbuilt projection system](../../setting-up-projections/). Therefore consider configuring your own Event Sourcing Repository only if you want  to build your own projecting system.
{% endhint %}

## Custom Event Sourcing Repository

We do start by implementing **EventSourcingRepository** interface:

```php
interface EventSourcedRepository
{
    1. public function canHandle(string $aggregateClassName): bool;
    
    2. public function findBy(string $aggregateClassName, array $identifiers, int $fromAggregateVersion = 1) :  EventStream;

    3. public function save(array $identifiers, string $aggregateClassName, array $events, array $metadata, int $versionBeforeHandling): void;
}
```

1. **canHandle** - Tells whatever given Aggregate is handled by this Repository
2. **findBy** - Method returns previously created events for given aggregate. Which Ecotone will use to reconstruct the Aggregate.&#x20;
3. **save** - Stores events recorded by Event Sourced Aggregate

and then we need to mark class which implements this interface as **Repository**

```php
#[Repository]
class CustomEventSourcingRepository
```

## Storing Events

Ecotone provides enough information to decide how to store provided events.&#x20;

```php
public function save(
    array $identifiers, 
    string $aggregateClassName, 
    array $events, 
    array $metadata, 
    int $versionBeforeHandling
): void;
```

Identifiers will hold array of **identifiers** related to given aggregate (e.g. _\["orderId" ⇒ 123]_). \
**Events** will be list of Ecotone's Event classes, which contains of **payload** and **metadata,** where payload is your Event class instance and metadata is specific to this event. \
**Metadata** as parameter is generic metadata available at the moment of Aggregate execution.\
**Version** before handling on other hand is the version of the Aggregate before any action was triggered on it. This can be used to protect from concurrency issues.

The structure of Events is as follows:

```php
class Event
{
    private function __construct(
        private string $eventName, // either class name or name of the event
        private object $payload, // event object instance
        private array $metadata // related metadata
    )
}
```

### Core metadata

It's worth to mention about Ecotone's Events and especially about metadata part of the Event.\
Each metadata for given Event contains of three core Event attributes:

**"\_aggregate\_id" -** This provides aggregate identifier of related Aggregate

**"\_aggregate\_version" -** This provides version of the related Event (e.g. 1/2/3/4)

**"\_aggregate\_type" -** This provides type of the Aggregate being stored, which can be customized

### Aggregate Type

If our repository stores multiple Aggregates is useful to have the information about the type of Aggregate we are storing. However keeping the class name is not best idea, as simply refactor would break our Event Stream. Therefore Ecotone provides a way to mark our Aggregate type using Attribute

```php
#[EventSourcingAggregate]
#[AggregateType("basket")]
class Basket
```

This now will be passed together with Events under **\_aggregate\_type** metadata.

### Named Events

In Ecotone we can name the events to avoid storing class names in the Event Stream, to do so we use NamedEvent.

```php
#[NamedEvent("order_was_placed")]
class OrderWasPlaced
```

then when events will be passed to save method, they will automatically provide this name under **eventName** property.

## Snapshoting

With custom repository we still can use inbuilt [Snapshoting mechanism](snapshoting.md). To use it for customized repository we will use **BaseEventSourcingConfiguration**.

```php
#[ServiceContext]
public function configuration()
{
    return BaseEventSourcingConfiguration::withDefaults()
            ->withSnapshotsFor(Basket::class, thresholdTrigger: 100);
}
```

Ecotone then after fetching snapshot, will load events only from this given moment using **\`fromAggregateVersion\`.**

```php
public function findBy(
    string $aggregateClassName, 
    array $identifiers, 
    int $fromAggregateVersion = 1
) 
```

## Testing

If you want to test out your flow and storing with your custom Event Sourced Repository, you should disable default in memory repository

```php
$repository = new CustomEventSourcingRepository;
$ecotoneLite = EcotoneLite::bootstrapFlowTesting(
    [OrderAggregate::class, CustomEventSourcingRepository::class],
    [CustomEventSourcingRepository::class => $repository],
    addInMemoryEventSourcedRepository: false,
);

$ecotoneLite->sendCommand(new PlaceOrder());

$this->assertNotEmpty($repository->getEvents());
```
