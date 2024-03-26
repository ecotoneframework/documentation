# Repository

## Business Repository Interface

Special type of `Business Interface` is `Repository`. \
If you want to fetch or store Aggregate register under [Ecotone's Repository](../repository.md) you may use                 `#[Repository]` attribute. &#x20;

### For State-Stored Aggregate

```php
interface OrderRepository
{
    #[Repository]
    public function getOrder(string $twitId): Order;

    #[Repository]
    public function findOrder(string $twitId): ?Order;

    #[Repository]
    public function save(Twitter $twitter): void;
}
```

Ecotone will read type hint to understand, which Aggregate you would like to fetch or save.

{% hint style="info" %}
Implementation will be delivered by Framework. All you need to do is to define the interface and it will available in your Dependency Container
{% endhint %}

### For Event Sourced Aggregate

```php
interface OrderRepository
{
    #[Repository]
    public function getOrder(string $twitId): Order;

    #[Repository]
    public function findOrder(string $twitId): ?Order;

    #[Repository]
    #[RelatedAggregate(Order::class)]
    public function save(string $aggregateId, int $currentVersion, array $events): void;
}
```

The difference is in `save` method, you need to provide `aggregate id, current aggregate's version and array of events` you would like to store.
