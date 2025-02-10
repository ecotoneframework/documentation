---
description: Asynchronous PHP Workers
---

# Lesson 6: Asynchronous Handling

{% hint style="info" %}
Not having code for _Lesson 6?_ \
\
`git checkout lesson-6`
{% endhint %}

`Ecotone` provides abstractions for asynchronous execution.

### Asynchronous

\
We got new requirement:\
`User should be able to place order for different products.`&#x20;

We will need to build `Order` aggregate.

Let's start by creating `PlaceOrderCommand` with ordered product Ids

```php
namespace App\Domain\Order;

class PlaceOrderCommand
{
    private int $orderId;
    /**
     * @var int[]
     */
    private array $productIds;

    /**
     * @return int[]
     */
    public function getProductIds(): array
    {
        return $this->productIds;
    }

    public function getOrderId() : int
    {
        return $this->orderId;
    }
}
```

We will need `OrderedProduct` value object, which will describe, cost and identifier of ordered product

```php
namespace App\Domain\Order;

class OrderedProduct
{
    private int $productId;

    private int $cost;

    public function __construct(int $productId, int $cost)
    {
        $this->productId = $productId;
        $this->cost = $cost;
    }

    public function getCost(): int
    {
        return $this->cost;
    }
}
```

And our `Order` aggregate

```php
namespace App\Domain\Order;

use App\Infrastructure\AddUserId\AddUserId;
use Ecotone\Messaging\Attribute\Asynchronous;
use Ecotone\Modelling\Attribute\Aggregate;
use Ecotone\Modelling\Attribute\AggregateIdentifier;
use Ecotone\Modelling\Attribute\CommandHandler;
use Ecotone\Modelling\Attribute\QueryHandler;
use Ecotone\Modelling\QueryBus;

#[Aggregate]
#[AddUserId]
class Order
{
    #[AggregateIdentifier]
    private int $orderId;

    private int $buyerId;

    /**
     * @var OrderedProduct[]
     */
    private array $orderedProducts;

    private function __construct(int $orderId, int $buyerId, array $orderedProducts)
    {
        $this->orderId = $orderId;
        $this->buyerId = $buyerId;
        $this->orderedProducts = $orderedProducts;
    }
    
    #[CommandHandler("order.place")]
    public static function placeOrder(PlaceOrderCommand $command, array $metadata, QueryBus $queryBus) : self
    {
        $orderedProducts = [];
        foreach ($command->getProductIds() as $productId) {
            $productCost = $queryBus->sendWithRouting("product.getCost", ["productId" => $productId]);
            $orderedProducts[] = new OrderedProduct($productId, $productCost->getAmount());
        }

        return new self($command->getOrderId(), $metadata["userId"], $orderedProducts);
    }

    #[QueryHandler("order.getTotalPrice")]
    public function getTotalPrice() : int
    {
        $totalPrice = 0;
        foreach ($this->orderedProducts as $orderedProduct) {
            $totalPrice += $orderedProduct->getCost();
        }

        return $totalPrice;
    }
}
```

`placeOrder` - Place order method make use of `QueryBus` to retrieve cost of each ordered product.\
You could find out, that we are not using `application/json` for `product.getCost` query, `ecotone/jms-converter` can handle `array` transformation, so we do not need to use `json`.

{% hint style="info" %}
You could inject service into  `placeOrder` that will hide `QueryBus` implementation from the domain, or you may get this data from `data store` directly. We do not want to complicate the solution now, so we will use `QueryBus` directly.&#x20;
{% endhint %}

{% hint style="success" %}
We do not need to change or add new `Repository`, as our exiting one can handle any new aggregate arriving in our system.

Let's change our testing class and run it!
{% endhint %}

```php
class EcotoneQuickstart
{
    private CommandBus $commandBus;
    private QueryBus $queryBus;

    public function __construct(CommandBus $commandBus, QueryBus $queryBus)
    {
        $this->commandBus = $commandBus;
        $this->queryBus = $queryBus;
    }

    public function run() : void
    {
        $this->commandBus->sendWithRouting(
            "product.register",
            ["productId" => 1, "cost" => 100]
        );
        $this->commandBus->sendWithRouting(
            "product.register",
            ["productId" => 2, "cost" => 300]
        );

        $orderId = 100;
        $this->commandBus->sendWithRouting(
            "order.place",
            ["orderId" => $orderId, "productIds" => [1,2]]
        );

        echo $this->queryBus->convertAndSend("order.getTotalPrice", MediaType::APPLICATION_X_PHP_ARRAY, ["orderId" => $orderId]);
    }
}
```

```php
bin/console ecotone:quickstart
Running example...
Start transaction
Product with id 1 was registered!
Commit transaction
Start transaction
Product with id 2 was registered!
Commit transaction
Start transaction
Commit transaction
400
Good job, scenario ran with success!
```

We want to be sure, that we do not lose any order, so we will register our `order.place Command Handler` to run asynchronously using `RabbitMQ` now. \
Let's start by adding extension to `Ecotone`, that can handle `RabbitMQ:`

```php
composer require ecotone/amqp
```

We also need to add our `ConnectionFactory` to our `Dependency Container.`&#x20;

{% tabs %}
{% tab title="Symfony - Local" %}
```php
# Add AmqpConnectionFactory in config/services.yaml

services:
    _defaults:
        autowire: true
        autoconfigure: true
    App\:
        resource: '../src/*'
        exclude: '../src/{Kernel.php}'
    Bootstrap\:
        resource: '../bootstrap/*'
        exclude: '../bootstrap/{Kernel.php}'

# You need to have RabbitMQ instance running on your localhost, or change DSN
    Enqueue\AmqpExt\AmqpConnectionFactory:
        class: Enqueue\AmqpExt\AmqpConnectionFactory
        arguments:
            - "amqp+lib://guest:guest@localhost:5672//"
```
{% endtab %}

{% tab title="Symfony - Docker" %}
```php
# Add AmqpConnectionFactory in config/services.yaml

services:
    _defaults:
        autowire: true
        autoconfigure: true
    App\:
        resource: '../src/*'
        exclude: '../src/{Kernel.php}'
    Bootstrap\:
        resource: '../bootstrap/*'
        exclude: '../bootstrap/{Kernel.php}'

# docker-compose.yml has RabbitMQ instance defined. It will be working without
# addtional configuration
    Enqueue\AmqpExt\AmqpConnectionFactory:
        class: Enqueue\AmqpExt\AmqpConnectionFactory
        arguments:
            - "amqp+lib://guest:guest@rabbitmq:5672//"
```
{% endtab %}

{% tab title="Laravel - Local" %}
```php
# Add AmqpConnectionFactory in bootstrap/QuickStartProvider.php

namespace Bootstrap;

use Illuminate\Support\ServiceProvider;
use Enqueue\AmqpExt\AmqpConnectionFactory;

class QuickStartProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton(AmqpConnectionFactory::class, function () {
            return new AmqpConnectionFactory("amqp+lib://guest:guest@localhost:5672//");
        });
    }
(...)
```
{% endtab %}

{% tab title="Laravel - Docker" %}
```php
# Add AmqpConnectionFactory in bootstrap/QuickStartProvider.php

namespace Bootstrap;

use Illuminate\Support\ServiceProvider;
use Enqueue\AmqpExt\AmqpConnectionFactory;

class QuickStartProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton(AmqpConnectionFactory::class, function () {
            return new AmqpConnectionFactory("amqp+lib://guest:guest@rabbitmq:5672//");
        });
    }
(...)
```
{% endtab %}

{% tab title="Lite - Local" %}
```php
# Add AmqpConnectionFactory in bin/console.php

// add additional service in container
public function __construct()
{
   $this->container = new Container();
   $this->container->set(Enqueue\AmqpExt\AmqpConnectionFactory::class, new Enqueue\AmqpExt\AmqpConnectionFactory("amqp+lib://guest:guest@localhost:5672//"));
}


```
{% endtab %}

{% tab title="Lite - Docker" %}
```php
# Add AmqpConnectionFactory in bin/console.php 

// add additional service in container
public function __construct()
{
   $this->container = new Container();
   $this->container->set(Enqueue\AmqpExt\AmqpConnectionFactory::class, new Enqueue\AmqpExt\AmqpConnectionFactory("amqp+lib://guest:guest@rabbitmq:5672//"));
}
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
We register our `AmqpConnectionFactory` under the class name `Enqueue\AmqpLib\AmqpConnectionFactory.` This will help Ecotone resolve it automatically, without any additional configuration.
{% endhint %}

Let's add our first `AMQP Backed Channel` (RabbitMQ Channel), in order to do it, we need to create our first `Application Context.` \
Application Context is a non-constructor class, responsible for extending `Ecotone` with extra configurations, that will help the framework act in a specific way. In here we want to tell `Ecotone` about `AMQP Channel` with specific name. \
Let's create new class `App\Infrastructure\MessagingConfiguration.`

```php
namespace App\Infrastructure;

class MessagingConfiguration
{
    #[ServiceContext]
    public function orderChannel()
    {
        return [
            AmqpBackedMessageChannelBuilder::create("orders")
        ];
    }
}
```

`ServiceContext` - Tell that this method returns configuration. It can return array of objects or a single object.

Now we need to tell our `order.place` Command Handler, that it should run asynchronously using our new`orders` channel.&#x20;

```php
use Ecotone\Messaging\Annotation\Asynchronous;

(...)

#[Asynchronous("orders")]
#[CommandHandler("order.place", endpointId: "place_order_endpoint")]
public static function placeOrder(PlaceOrderCommand $command, array $metadata, QueryBus $queryBus) : self
{
    $orderedProducts = [];
    foreach ($command->getProductIds() as $productId) {
        $productCost = $queryBus->sendWithRouting("product.getCost", ["productId" => $productId]);
        $orderedProducts[] = new OrderedProduct($productId, $productCost->getAmount());
    }

    return new self($command->getOrderId(), $metadata["userId"], $orderedProducts);
}
```

We do it by adding `Asynchronous` annotation with `channelName` used for asynchronous endpoint. \
Endpoints using `Asynchronous` are required to have `endpointId` defined, the name can be anything as long as it's not the same as `routing key (order.place)`.&#x20;

```php
#[CommandHandler("order.place", endpointId: "place_order_endpoint")]
```

{% hint style="info" %}
You may mark [`Event Handler`](broken-reference) as asynchronous the same way.
{% endhint %}

{% hint style="success" %}
Let's run our command which will tell us what asynchronous endpoints we have defined in our system: `ecotone:list`
{% endhint %}

```php
bin/console ecotone:list
+--------------------+
| Endpoint Names     |
+--------------------+
| orders             |
+--------------------+
```

We have new asynchronous endpoint available `orders.` Name comes from the message channel name.\
You may wonder why it is not `place_order_endpoint,` it's because via single asynchronous channel we can handle multiple endpoints, if needed. This is further explained in [asynchronous section](../modelling/asynchronous-handling/scheduling.md).

Let's change `orderId` in our testing command, so we can place new order.

```php
public function run() : void
{
    $this->commandBus->sendWithRouting(
        "product.register",
        ["productId" => 1, "cost" => 100]
    );
    $this->commandBus->sendWithRouting(
        "product.register",
        ["productId" => 2, "cost" => 300]
    );

    $orderId = 990;
    $this->commandBus->sendWithRouting(
        "order.place",
        ["orderId" => $orderId, "productIds" => [1,2]]
    );

    echo $this->queryBus->sendWithRouting("order.getTotalPrice", ["orderId" => $orderId]);
}
```

After running our testing command `bin/console ecotone:quickstart`we should get an exception:

```php
AggregateNotFoundException:
                                                                               
  Aggregate App\Domain\Order\Order:getTotalPrice was not found for indentifie  
  rs {"orderId":990}  
```

That's fine, we have registered `order.place` Command Handler to run asynchronously, so we need to run our `asynchronous endpoint` in order to handle `Command Message`. If you did not received and exception, it's probably because `orderId` was not changed and we already registered such order.\
\
Let's run our asynchronous endpoint

```php
bin/console ecotone:run orders --handledMessageLimit=1 --stopOnFailure -vvv
[info] {"orderId":990,"productIds":[1,2]}
```

Like we can see, it ran our Command Handler and placed the order.\
We can change our testing command to run only `Query Handler`and check, if the order really exists now.

```php
class EcotoneQuickstart
{
    private CommandBus $commandBus;
    private QueryBus $queryBus;

    public function __construct(CommandBus $commandBus, QueryBus $queryBus)
    {
        $this->commandBus = $commandBus;
        $this->queryBus = $queryBus;
    }

    public function run() : void
    {
        $orderId = 990;

        echo $this->queryBus->sendWithRouting("order.getTotalPrice", ["orderId" => $orderId]);
    }
}
```

```php
bin/console ecotone:quickstart -vvv
Running example...
400
Good job, scenario ran with success!
```

There is one thing we can change. \
As in asynchronous scenario we may not have access to the context of executor to enrich the message,, we can change our `AddUserIdService Interceptor` to perform the action before sending it to asynchronous channel.\
This Interceptor is registered as `Before Interceptor` which is before execution of our Command Handler, but what we want to achieve is, to call this interceptor before message will be send to the asynchronous channel. For this there is `Presend` Interceptor available.\
Change `Before` annotation to `Presend` annotation and we are done.

```php
namespace App\Infrastructure\AddUserId;

class AddUserIdService
{
   #[Presend(0, AddUserId::class, true)]
    public function add() : array
    {
        return ["userId" => 1];
    }
}
```

{% hint style="success" %}
Ecotone will do it best to handle serialization and deserialization of your headers.&#x20;
{% endhint %}

Now if non-administrator will try to execute this, exception will be thrown, before the Message will be put to the asynchronous channel. Thanks to `Presend` interceptor, we can validate messages, before they will go asynchronous, to prevent sending incorrect messages.

{% hint style="info" %}
The final code is available as lesson-7:\
\
`git checkout lesson-7`
{% endhint %}

{% hint style="success" %}
We made it through, Congratulations! \
We have successfully registered asynchronous Command Handler and safely placed the order. \
\
We have finished last lesson. You may now apply the knowledge in real project or check more advanced usages starting here [Modelling Overview](../modelling/message-driven-php-introduction.md).
{% endhint %}
