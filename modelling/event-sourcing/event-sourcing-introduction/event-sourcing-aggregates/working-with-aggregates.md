# Working with Aggregates

## Working with Event Sourcing Aggregates

Just as with Standard Aggregate, ES Aggregates are called by Command Handlers, however what they return are Events and they do not change their internal state.

```php
#[EventSourcingAggregate]
class Product
{
    use WithAggregateVersioning;

    #[Identifier]
    private string $id;

    #[CommandHandler]
    public static function create(CreateProduct $command) : array
    {
        return [new ProductWasCreated($command->id, $command->name, $command->price)];
    }
}
```

When this Aggregate will be called via Command Bus with **CreateProduct** Command, it will then return new **ProductWasCreated** Event.&#x20;

<figure><img src="../../../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

{% hint style="success" %}
Command Handlers may return single events, multiple events or no events at all, if nothing is meant to be changed.
{% endhint %}

## Event Stream

Aggregates under the hood make use of Partition persistence strategy (Refer to [Working with Event Streams](../working-with-event-streams.md)). This means that we need to know:

* Aggregate Version
* Aggregate Id
* Aggregate Type

### Aggregate Version

To find out about current version of Aggregate Ecotone will look for property marked with **Version Attribute**.

```php
#[Version]
private int $version = 0;
```

We don't to add this property directly, we can use trait instead:

```php
#[EventSourcingAggregate]
class Product
{
    use WithAggregateVersioning;
```

Anyways, this is all we need to do, as Ecotone will take care of reading and writing to this property. \
This way we can focus on the business logic of the Aggregate, and Framework will take care of tracking the version.

### Aggregate Id (Partition Key)

We need to tell to Ecotone what is the Identifier of our Event Sourcing Aggregate.  \
This is done by having property marked with Identifier in the Aggregate:

```php
#[Identifier]
private string $id;
```

As Command Handlers are pure and do not change the state of our Event Sourcing Aggregate, this means we need a different way to mutate the state in order to assign the identifier.\
For changing the state we use **EventSourcingHandler** attribute, which tell Ecotone that if given Event happens, then trigger this method afterwards:

```php
#[EventSourcingHandler]
public function applyProductWasCreated(ProductWasCreated $event) : void
{
    $this->id = $event->id;
}
```

We will explore how applying Events works more in [next section](applying-events.md).

### Aggregate Type

Aggregate Type will be the same as Aggregate Class. \
We can decouple the class from the Aggregate Type, more about this can be found in "[Making Stream immune to changes](../persistence-strategy/making-stream-immune-to-changes.md)" section.

### Recording Events in the Event Stream

So when this Command Handler happens:

```php
#[CommandHandler]
public static function create(CreateProduct $command) : array
{
    return [new ProductWasCreated($command->id, $command->name, $command->price)];
}
```

What actually will happen under the hood is that this Event will be applied to the Event Stream:

```php
$eventStore->appendTo(
    Product::class, // Stream name
    [
        Event::create(
            $event,
            metadata: [
                '_aggregate_id' => 1,
                '_aggregate_version' => 1,
                '_aggregate_type' => Product::class,
            ]
        )
    ]
);
```

As storing in Event Store is abstracted away, the code stays clean and contains only of the business part. \
We can [customize](../persistence-strategy/making-stream-immune-to-changes.md) the Stream Name, Aggregate Type and even Event Names when needed.
