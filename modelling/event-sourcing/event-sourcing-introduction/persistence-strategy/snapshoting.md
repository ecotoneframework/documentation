---
description: PHP Event Sourcing Snapshoting
---

# Snapshoting

In general having streams in need for snapshots may indicate that our model needs revisiting. \
We may cut the stream on some specific event and begin new one, like at the end of month from all the transactions we generate invoice and we start new stream for next month. \
\
However if cutting the stream off is not an option for any reason, we can use snapshots to avoid loading all events history for given Aggregate. Every given set of events snapshot will be taken, stored and retrieved on next calls, to fetch only events that happened after that.

## Setting up&#x20;

`EventSourcingConfiguration` provides the following interface to set up snapshots.

```php
class EventSourcingConfiguration
{
    public const DEFAULT_SNAPSHOT_TRIGGER_THRESHOLD = 100;

    public function withSnapshotsFor(
        string $aggregateClassToSnapshot, // 1.
        int $thresholdTrigger = self::DEFAULT_SNAPSHOT_TRIGGER_THRESHOLD, // 2.
        string $documentStore = DocumentStore::class // 3.
    ): static
}
```

1. `$aggregateClassToSnapshot` - class name of an aggregate you want Ecotone to save snapshots of
2. `$thresholdTrigger` - amount of events for interval of taking a snapshot
3. `$documentStore` - reference to document store which will be used for saving/retrieving snapshots

To set up snapshots we will define [ServiceContext](../../../../messaging/service-application-configuration.md) configuration.

```php
#[ServiceContext]
public function aggregateSnapshots()
{
    return EventSourcingConfiguration::createWithDefaults()
            ->withSnapshotsFor(Ticket::class, 1000)
            ->withSnapshotsFor(Basket::class, 500, MyDocumentStore::class)
    ;
}
```

{% hint style="info" %}
Ecotone make use of [Document Store](../../../../messaging/document-store.md) to store snapshots, by default it's enabled with event-sourcing package. \
\
If you want to clean the snapshots, you can do it manually. Snapshots are stored in `aggregate_snapshots` collection.
{% endhint %}

## Snapshot threshold

Threshold states at which interval snapshots should be done. Therefore with below configuration:

```php
->withSnapshotsFor(Ticket::class, 500)
```

snapshots will be done every 500 events. Then when snapshot will be loaded, it will start loading the events from event number 501 for given Aggregate instance.
