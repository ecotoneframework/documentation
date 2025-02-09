# Making Stream immune to changes

Changes in the Application will happen. After some time we may want to refactor namespaces, change the name of Aggregate or an Event. However those kind of changes may break our system, if we already have production data which references to any of those. \
Therefore to make our Application to immune to future changes we need a way to decouple the code from the data in the storage, and this is what Ecotone provides.

## Custom Stream Name

Our Event Stream name by default is based on the Aggregate Class name. Therefore to make it immune to changes we may provide custom Stream Name. To do it we will apply `Stream` attribute to the aggregate:

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

{% hint style="info" %}
You may wonder what is the difference between Stream name and Aggregate Type. \
By default the are the same, however we could use the same Stream name between different Aggregates, to store them all together within same Table. \
\
This may useful during migration to next version of the Aggregate, where we would want to hold both versions in same Stream.
{% endhint %}

## Storing Events By Names

To avoid storing class names of Events in the `Event Store` we may mark them with name:

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

This way Ecotone will do the mapping before storing an Event and when retrieving the Event in order to deserialize it to correct class.

## Testing

It's worth to remember that if we want test storing Events using provided Event Named, we need to add them under recognized classes, so Ecotone knows that should scan those classes for Attributes:

```php
$ecotoneLite = EcotoneLite::bootstrapFlowTesting(
    [Basket::class, BaskeWasCreated::class],
);
```
