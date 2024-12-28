# Business Repository

## Business Repository Interface

Special type of **Business Interface** is **Repository**. \
This Interface allows us to simply load and store our Aggregates directly. In situations when we call Command directly in our Aggregates we won't be in need to use it. However for some specific cases, where we need to load Aggregate and store it outside of Aggregate's Command Handler, this business interface becomes useful.&#x20;

{% hint style="info" %}
To make use of this Business Interface, we need our [Aggregate Repository](../repository.md) being registered.
{% endhint %}

## Business Repository

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

Ecotone will read type hint to understand which Aggregate you would like to fetch or save.

{% hint style="info" %}
Implementation will be delivered by Ecotone. All you need to do is to define the interface and it will available in your Dependency Container
{% endhint %}

## Pure Event Sourced Repository <a href="#for-event-sourced-aggregate" id="for-event-sourced-aggregate"></a>

When using Pure Event Sourced Aggregate, instance of Aggregate does not hold recorded Events. Therefore passing aggregate instance would not contain any information about recorded events. \
For Pure Event Sourced Aggregates, we can use direct event passing to the repository:

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
