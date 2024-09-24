# Persistence Strategies

## Persistence Strategy

Describes how streams with events will be stored.\
Each Event Stream is separate Database Table, yet how those tables are created and what are constraints do they protect depends on the persistence strategy.

## Simple Stream Strategy

This is the basics Stream Strategy which involves no constraints. This means that we can append any Events to it without providing any additional metadata.&#x20;

```php
$eventStore->create($streamName, streamMetadata: [
    "_persistence" => 'simple',
]);

$eventStore->appendTo(
    $streamName,
    [
        Event::create(
            payload: new TicketWasRegistered('123', 'Johnny', 'alert'),
            metadata: [
                'executor' => 'johnny',
            ]
        )
    ]
);
```

Now as this is free append involves no could re-run this code apply exactly the same Event. \
This can sounds silly, but it 's make it useful for particular cases. \
It make it super easy to append new Events. We basically could just add this action in our code and keep applying Events to the Event Stream, we don't need to know context of what happened before.&#x20;

This is useful for scenarios where we just want to store information without putting any business logic around this. \
It could be used to continues stream of information like:&#x20;

* Temperature changes
* Counting car passing by in traffic jam
* Recording clicks and user views. &#x20;

## Partition Stream Strategy

This the default persistence strategy. It does creates partitioning within Event Stream to ensure that we always maintain correct history within partition. \
This way we can be sure that each Event contains details on like Aggregate id it does relate to, on which version it was applied, to what Aggregate it references to.

```php
$eventStore->create($streamName, streamMetadata: [
    "_persistence" => 'partition',
]);
```

```php
$eventStore->appendTo(
    $streamName,
    [
        Event::create(
            new TicketWasRegistered('123', 'Johnny', 'alert'),
            [
                '_aggregate_id' => 123,
                '_aggregate_version' => 1,
                '_aggregate_type' => 'ticket',
            ]
        )
    ]
);
```

The tricky part here is that we need to know Context in order to apply the Event, as besides the Aggregate Id, we need to provide Version. To know the version we need to be aware of last previous applied Event.&#x20;

{% hint style="success" %}
When this persistence strategy is used with Ecotone's Aggregate, Ecotone resolve metadata part on his own, therefore working with this Stream becomes easy. However when working directly with Event Store getting the context may involve extra work.&#x20;
{% endhint %}

This Stream Strategy is great whenever business logic is involved that need to be protected. \
This solves for example the problem of concurrent access on the database level, as we for example can't store Event for same Aggregate Id and Version twice in the Event Stream. \
\
We would use it in most of business scenarios where knowing previous state in order to make the decision is needed, like:

* Check if we can change Ticket based on status
* Performing invocing from previous transactions
* Decide if Order can be shipped

{% hint style="info" %}
This is the default persistence strategy used whenever we don't specify otherwise.
{% endhint %}

## Stream Per Aggregate Strategy

This is similar to Partition strategy, however each Partition is actually stored in separate Table, instead of Single One.&#x20;

```php
$eventStore->create($streamName, streamMetadata: [
    "_persistence" => 'aggregate',
]);
```

```php
$eventStore->appendTo(
    $streamName,
    [
        Event::create(
            new TicketWasRegistered('123', 'Johnny', 'alert'),
            [
                '_aggregate_id' => 123,
                '_aggregate_version' => 1,
                '_aggregate_type' => 'ticket',
            ]
        )
    ]
);
```

This can be used when amount of partitions is really low and volume of events within partition is huge.&#x20;

{% hint style="info" %}
Take under consideration that each aggregate instance will have separate table. When this strategy is used with a lot of Aggregate instance, the volume of tables in the database may become hard to manage.&#x20;
{% endhint %}

## Custom Strategy

You may provide your own Customer Persistence Strategy as long as it implements `PersistenceStrategy`.

```php
#[ServiceContext]
public function aggregateStreamStrategy()
{
    return EventSourcingConfiguration::createWithDefaults()
        ->withCustomPersistenceStrategy(new CustomStreamStrategy(new FromProophMessageToArrayConverter()));
}
```

## Setting global Persistence Strategy

To set given persistence strategy as default, we can use ServiceContext:

```php
#[ServiceContext]
public function persistenceStrategy()
{
    return EventSourcingConfiguration::createWithDefaults()
        ->withSimpleStreamPersistenceStrategy();
}
```

## Multiple Persistence Strategies

Once set, the persistence strategy will apply to all streams in your application. However, you may face a situation when you need to have a different strategy for one or more of your streams.

```php
#[ServiceContext]
public function eventSourcingConfiguration(): EventSourcingConfiguration
{
    return EventSourcingConfiguration::createWithDefaults()
        ->withPersistenceStrategyFor('some_stream', LazyProophEventStore::AGGREGATE_STREAM_PERSISTENCE)
    ;
}
```

The above will make the Simple Stream Strategy as default however, for `some_stream` Event Store will use the Aggregate Stream Strategy.

{% hint style="warning" %}
Be aware that we won't be able to set Custom Strategy that way.
{% endhint %}
