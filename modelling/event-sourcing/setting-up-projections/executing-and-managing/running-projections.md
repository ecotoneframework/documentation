---
description: PHP Event Sourcing Projections
---

# Running Projections

## Running Projection

### Synchronously Event Driven Projection

By default `Ecotone` runs the projections synchronously with your aggregate changes.\
This kind of running configuration can be used to avoid eventual consistency problems or for testing purposes. \
However when you expect multiple accesses to your Aggregates at the same time, you may consider asynchronous projection to protect yourself from concurrency problems.

{% hint style="success" %}
This projections are running within same transaction as your Event Store changes. \
This will ensure atomic consistency between your aggregate and projection.
{% endhint %}

### Polling Projection

You may run your projection in the background. \
It will query the database within constant time intervals, to look if new events have been registered. \
Each projection is running as **separate process**. \
To register Polling Projection make use of [ServiceContext](../../../../messaging/service-application-configuration.md).

```php
#[ServiceContext]
public function basketList()
{
    return ProjectionRunningConfiguration::createPolling("basket_list");
}
```

Then we can run:

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:run basket_list -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:run basket_list -vvv
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->run("basket_list");
```
{% endtab %}
{% endtabs %}

### Asynchronously Event Driven Projection

You may pass your projections in event driven manner using [asynchronous channels](../../../asynchronous-handling/).

```php
#[Asynchronous("asynchronous_projections")]
#[Projection("basket_list")]
class BasketList
```

\
The difference between `Polling` and `Event Driven` projection is the way they are triggered. \
The `Event Driven` is only **triggered when new event comes to the system**. This avoid the pitfall of continues database access while using `Polling Projection`.\
The second strength of Asynchronously Event Driven Projection is possibility of registering multiple projections under same channel (which is same consumer).

## Custom Configuration

### **Event Sourcing Customer Configuration**

You may customize your Event Sourcing configuration with following configuration:

```php
class ProjectionConfiguration
{
    #[ServiceContext]
    public function configureEventSourcing()
    {
        return [
            EventSourcingConfiguration::createWithDefaults()
                ->withEventStreamTableName('event_stream') // name of event stream table
                ->withProjectionsTableName('projections') // name of projection table
                ->withInitializeEventStoreOnStart(false) // will check and create above tables if needed
                ->withLoadBatchSize(1) // amount of events loaded on each projection run,
        ];
    }
}

```

### Projection Custom Configuration

You may configure your projection custom configuration.\
Take under consideration that some configuration may have sense only in case of [Polling Projection](running-projections.md#polling-projection).

```php
#[ServiceContext]
public function projectionConfiguration()
{
    return ProjectionRunningConfiguration::createPolling("ticket_list")
            ->withOption("load_count", 10000);
}
```

**load\_count** //Default: null

Change load batch size in each run for single projection.&#x20;

{% hint style="info" %}
You can use `load_count` option in optimization maner. When defined, projection will load events in batches. This can come in handy especially when you want to reset projection on stream with large amount of events.
{% endhint %}

**cache\_size** //Default: 1000

The cache size is how many stream names are cached in memory, the higher the number the less queries are executed and therefore the projection runs faster, but it consumes more memory.

**sleep** //Default: 100000

The sleep options tells the projection to sleep that many microseconds before querying the event store again when no events were found in the last trip. This reduces the number of cpu cycles without the projection doing any real work.

**persist\_block\_size** //Default: 1000

The persist block size tells the projector to persist its changes after a given number of operations. This increases the speed of the projection a lot. When you only persist every 1000 events compared to persist on every event, then 999 write operations are saved. The higher the number, the fewer write operations are made to your system, making the projections run faster. On the other side, in case of an error, you need to redo the last operations again. If you are publishing events to the outside world within a projection, you may think of a persist block size of 1 only.

**lock\_timeout\_ms** //Default: 1000

Indicates the time (in milliseconds) the projector is locked. During this time no other projector with the same name can be started. A running projector will update the lock timeout on every loop, except you configure an update lock threshold.

**update\_lock\_threshold** //Default: 0

If update lock threshold is set to a value greater than 0 the projection won't update lock timeout until number of milliseconds have passed. Let's say your projection has a sleep interval of 100 ms and a lock timeout of 1000 ms. By default the projector updates lock timeout after each run so basically every 100 ms the lock timeout is set to: `now() + 1000 ms` This causes a lot of extra work for your database and in case the database is replicated this can cause a lot of network traffic, too.

**gap\_detection** //Default: `new \Prooph\EventStore\Pdo\Projection\GapDetection()`

Gap Detection makes projection to wait for upcoming events if any gap occurs in your event stream. To disable Gap Detection you can set value to `null`.
