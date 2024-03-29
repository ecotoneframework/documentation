---
description: DDD PHP
---

# Lesson 2: Tactical DDD

{% hint style="info" %}
Not having code for _Lesson 2?_&#x20;

`git checkout lesson-2`
{% endhint %}

### Aggregate

An Aggregate is an entity or group of entities that is always kept in a consistent state.  \
Aggregates are very explicitly present in the Command Model, as that is where change is initiated and business behaviour is placed.

Let's create our first _Aggregate_ `Product.`

```php
namespace App\Domain\Product;

use Ecotone\Modelling\Attribute\Aggregate;
use Ecotone\Modelling\Attribute\Identifier;
use Ecotone\Modelling\Attribute\CommandHandler;
use Ecotone\Modelling\Attribute\QueryHandler;

#[Aggregate]
class Product
{
    #[Identifier]
    private int $productId;

    private int $cost;

    private function __construct(int $productId, int $cost)
    {
        $this->productId = $productId;
        $this->cost = $cost;
    }

    #[CommandHandler]
    public static function register(RegisterProductCommand $command) : self
    {
        return new self($command->getProductId(), $command->getCost());
    }

    #[QueryHandler]
    public function getCost(GetProductPriceQuery $query) : int
    {
        return $this->cost;
    }
}
```

1. **Aggregate** attribute marks class to be known as Aggregate
2. **Identifier** marks properties as identifiers of specific Aggregate instance. Each _Aggregate_ must contains at least one identifier.
3. **CommandHandler** enables command handling on specific method just as we did in [Lesson 1](php-messaging-architecture.md). \
   If method is static, it's treated as a [factory method](https://en.wikipedia.org/wiki/Factory\_method\_pattern) and must return a new aggregate instance. Rule applies as long as we use [State-Stored Aggregate](../modelling/command-handling/state-stored-aggregate/#state-stored-aggregate) instead of [Event Sourcing Aggregate](broken-reference).
4. **QueryHandler** enables query handling on specific method just as we did in Lesson 1.

{% hint style="info" %}
If you want to known more details about _Aggregate_ start with chapter [State-Stored Aggregate](../modelling/command-handling/state-stored-aggregate/#state-stored-aggregate)
{% endhint %}

Now remove `App\Domain\Product\ProductService` as it contains handlers for the same command and query classes. \
Before we will run our test scenario, we need to register `Repository`.

{% hint style="info" %}
Usually you will mark `services` as Query Handlers not `aggregates` .However Ecotone does not block possibility to place Query Handler on _Aggregate_. It's up to you to decide.
{% endhint %}

### Repository

Repositories are used for retrieving and saving the aggregate to persistent storage. \
We will build an in-memory implementation for now.

```php
namespace App\Domain\Product;

use Ecotone\Modelling\Attribute\Repository;
use Ecotone\Modelling\StandardRepository;

 #[Repository] // 1
class InMemoryProductRepository implements StandardRepository // 2
{
    /**
     * @var Product[]
     */
    private $products = [];

    // 3
    public function canHandle(string $aggregateClassName): bool
    {
        return $aggregateClassName === Product::class;
    }

    // 4
    public function findBy(string $aggregateClassName, array $identifiers): ?object
    {
        if (!array_key_exists($identifiers["productId"], $this->products)) {
            return null;
        }

        return $this->products[$identifiers["productId"]];
    }

    // 5
    public function save(array $identifiers, object $aggregate, array $metadata, ?int $expectedVersion): void
    {
        $this->products[$identifiers["productId"]] = $aggregate;
    }
}
```

1. **Repository** attribute marks class to be known to `Ecotone` as Repository.
2. We need to implement some methods in order to allow `Ecotone` to retrieve and save Aggregate. Based on implemented interface, `Ecotone` knowns, if _Aggregate_ is state-stored or event sourced.
3. **canHandle** tells which classes can be handled by this specific repository.
4. **findBy**  return found aggregate instance or null. As there may be more, than single indentifier per aggregate, identifiers are array.
5. **save** saves an aggregate instance. You do not need to bother right what is `$metadata` and `$expectedVersion`.

{% hint style="info" %}
If you want to known more details about _Repository_ start with chapter [Repository](../modelling/command-handling/repository.md)
{% endhint %}

{% tabs %}
{% tab title="Laravel" %}
```php
# As default auto wire of Laravel creates new service instance each time 
# service is requested from Depedency Container, we need to register 
# ProductService as singleton.

# Go to bootstrap/QuickStartProvider.php and register our ProductService

namespace Bootstrap;

use App\Domain\Product\InMemoryProductRepository;
use Illuminate\Support\ServiceProvider;

class QuickStartProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton(InMemoryProductRepository::class, function(){
            return new InMemoryProductRepository();
        });
    }
(...)
```
{% endtab %}

{% tab title="Symfony" %}
```php
Everything is set up by the framework, please continue...
```
{% endtab %}

{% tab title="Lite" %}
```
Everything is set up, please continue...
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Let's run our testing command:

```php
bin/console ecotone:quickstart
Running example...
100
Good job, scenario ran with success!
```
{% endhint %}

Have you noticed what we are missing here? Our `Event Handler` was not called, as we do not publish the `ProductWasRegistered` event anymore.

### Event Publishing

In order to automatically publish events recorded within Aggregate, we need to add method annotated with `AggregateEvents.` This will tell `Ecotone` where to get the events from.\
\
`Ecotone` comes with default implementation, that can be used as trait **WithEvents**.

```php
use Ecotone\Modelling\WithEvents;

#[Aggregate]
class Product
{
    use WithEvents;

    #[Identifier]
    private int $productId;

    private int $cost;

    private function __construct(int $productId, int $cost)
    {
        $this->productId = $productId;
        $this->cost = $cost;

        $this->recordThat(new ProductWasRegisteredEvent($productId));
    }
(...)
```

{% hint style="info" %}
You may implement your own method for returning events, if you do not want to be coupled with the framework.
{% endhint %}

{% hint style="success" %}
Let's run our testing command:
{% endhint %}

```php
bin/console ecotone:quickstart
Running example...
Product with id 1 was registered!
100
Good job, scenario ran with success!
```

{% hint style="success" %}
Congratulations, we have just finished Lesson 2.\
In this lesson we have learnt how to make use of Aggregates and Repositories.\
\
Now we will learn about Converters and Metadata
{% endhint %}
