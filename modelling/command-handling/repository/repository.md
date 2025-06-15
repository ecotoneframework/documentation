# Fetching/Storing Aggregates

## Default flow

In default flow there is no need to fetch or store Aggregates, because this is done for us. We simply need to **trigger an Command via CommandBus**. However in some cases, you may want to retake orchestration flow and do it directly. For that cases **Business Repository Interface** or **Instant Fetch Aggregate** can help you.

## Business Repository Interface

Special type of [**Business Interface**](../business-interface/) is **Repository**. \
This Interface allows us to simply load and store our Aggregates directly. In situations when we call Command directly in our Aggregates we won't be in need to use it. However for some specific cases, where we need to load Aggregate and store it outside of Aggregate's Command Handler, this business interface becomes useful.&#x20;

{% hint style="info" %}
To make use of this Business Interface, we need our [Aggregate Repository](./) being registered.
{% endhint %}

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

{% hint style="success" %}
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

## Instant Fetch Aggregate&#x20;

In cases where we want to fetch Aggregate directly without introducing orchestration into our code base or depending on the Repositories, we can use Fetch Aggregate Attribute.

{% hint style="success" %}
Instant Fetch Aggregate is available as part of **Ecotone Enterprise.**
{% endhint %}

To do instant fetch of Aggregate we will be using **Fetch** Attribute.\
Suppose we want PlaceOrder Command Handler, and we want to fetch User Aggregate:

```php
#[CommandHandler]
public function placeOrder(
    PlaceOrder $command,
    #[Fetch("payload.userId")] User $user
): void {
    // do something    
}
```

Fetch using [expression language](https://symfony.com/doc/current/reference/formats/expression_language.html) to evaluate the expression given inside the Attribute. \
For example having above "payload.userId" and following Command:

```php
class readonly PlaceOrder
{
    public function __construct(
        public string $orderId,
        public string $userId,
        public string $productId
    ) {
    }
```

Ecotone will use userId from the Command to fetch User Aggregate instance. \
&#xNAN;**"payload" is special variable within expression that points to our Command**, therefore whatever is available within the Command is available for us to do the fetching.\
This provides quick way of accessing related Aggregates without the need to inject Repositories.

### Allowing non existing Aggregates

By default Ecotone will throw Exception if Aggregate is not found, we can change the behaviour simply by allowing nulls in our method declaration:

```php
#[CommandHandler]
public function placeOrder(
    PlaceOrder $command,
    #[Fetch("payload.userId")] ?User $user // we marked it as possible null
): void {
    // do something    
}
```

### Accessing Message Headers

We can also use Message Headers to fetch our related Aggregate instance:

```php
#[CommandHandler]
public function placeOrder(
    PlaceOrder $command,
    #[Fetch("headers['userId']")] User $user
): void {
    // do something    
}
```

### Using External Services

In some cases we may not have enough information to provide correct Identifier, for example that may require some mapping in order to get the Identifier. For this cases we can use "reference" function to access any Service from Depedency Container in order to do the mapping.

```php
#[CommandHandler]
public function placeOrder(
    PlaceOrder $command,
    #[Fetch("reference('emailToIdMapper').map(payload.email)")] User $user
): void {
    // do something    
}
```
