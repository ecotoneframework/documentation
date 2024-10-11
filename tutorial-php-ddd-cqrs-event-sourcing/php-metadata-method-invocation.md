---
description: PHP Metadata and Method Invocation
---

# Lesson 4: Metadata and Method Invocation

{% hint style="info" %}
Not having code for _Lesson 4?_ \
\
`git checkout lesson-4`
{% endhint %}

### Metadata

Message can contain of Metadata. Metadata is just additional information stored along side to the Message's payload. It may contain things like **currentUser**, **timestamp**, **contentType**, **messageId**.&#x20;

{% hint style="info" %}
In **Ecotone** headers and metadata means the same. Those terms will be used interchangeably.
{% endhint %}

\
To test out Metadata, let's assume we just got new requirement for our Products in Shopping System.:

> User who registered the product, should be able to change it's price.&#x20;

Let's start by adding **ChangePriceCommand**

```php
namespace App\Domain\Product;

class ChangePriceCommand
{
    private int $productId;

    private Cost $cost;

    public function getProductId() : int
    {
        return $this->productId;
    }

    public function getCost() : Cost
    {
        return $this->cost;
    }
}
```

We will handle this Command in a minute. Let's first add user information for registering the product.\
We will do it, using **Metadata**. Let's get back to our Testing Class **EcotoneQuickstart** and add 4th argument to our **CommandBus** call.

```php
public function run() : void
{
    $this->commandBus->sendWithRouting(
        "product.register",
        \json_encode(["productId" => 1, "cost" => 100]),
        "application/json",
        metadata: [
            "userId" => 1
        ]
    );
            
    echo $this->queryBus->sendWithRouting("product.getCost", \json_encode(["productId" => 1]), "application/json");
}
```

**sendWithRouting** accepts 4th argument, which is **associative array.** Whatever we will place in here, will be available during message handling for us - This actually our Metadata. It's super simple to pass new Headers, it's matter of adding another key to the array.\
\
Now we can change our **Product** aggregate:&#x20;

```php
#[Aggregate]
class Product
{
    use WithAggregateEvents;

    #[Identifier]
    private int $productId;

    private Cost $cost;

    private int $userId;

    private function __construct(int $productId, Cost $cost, int $userId)
    {
        $this->productId = $productId;
        $this->cost = $cost;
        $this->userId = $userId;

        $this->recordThat(new ProductWasRegisteredEvent($productId));
    }

    #[CommandHandler("product.register")]
    public static function register(RegisterProductCommand $command, array $metadata) : self
    {
        return new self(
            $command->getProductId(), 
            $command->getCost(), 
            // all metadata is available for us. 
            // Ecotone automatically inject it, if second param is array
            $metadata["userId"]
        );
    }
```

We have added second parameter **$metadata** to our **CommandHandler**. **Ecotone** read parameters and evaluate what should be injected. We will see soon, how can we take control of this process. \
\
We can add **changePrice** method now to our Aggregate:

```php
#[CommandHandler("product.changePrice")]
public function changePrice(ChangePriceCommand $command, array $metadata) : void
{
    if ($metadata["userId"] !== $this->userId) {
        throw new \InvalidArgumentException("You are not allowed to change the cost of this product");
    }

    $this->cost = $command->getCost();
}
```

And let's call it with incorrect **userId** and see, if we get the exception.

```php
public function run() : void
{
    $this->commandBus->sendWithRouting(
        "product.register",
        \json_encode(["productId" => 1, "cost" => 100]),
        "application/json",
        [
            "userId" => 5
        ]
    );

    $this->commandBus->sendWithRouting(
        "product.changePrice",
        \json_encode(["productId" => 1, "cost" => 110]),
        "application/json",
        [
            "userId" => 3
        ]
    );        
```

{% hint style="success" %}
Let's run our testing command:
{% endhint %}

```php
bin/console ecotone:quickstart
Running example...
Product with id 1 was registered!

InvalidArgumentException
                                                          
  You are not allowed to change the cost of this product 
```

### Method Invocation

We have been just informed, that customers are registering new products in our system, which should not be a case. Therefore our next requirement is:

> Only administrator should be allowed to register new Product

Let's create simple **UserService** which will tell us, if specific user is administrator. \
In our testing scenario we will suppose, that only user with `id` of 1 is administrator.&#x20;

```php
namespace App\Domain\Product;

class UserService
{
    public function isAdmin(int $userId) : bool
    {
        return $userId === 1;
    }
}
```

Now we need to think where we should call our **UserService**. \
The good place for it, would not allow for any invocation of **product.register** command without being administrator, otherwise our constraint may be bypassed.\
**Ecotone** does allow for auto-wire like injection for endpoints. All services registered in Depedency Container are available.

```php
#[CommandHandler("product.register")]
public static function register(
    RegisterProductCommand $command, 
    array $metadata, 
    // Any non first class argument, will be considered an DI Service to inject
    UserService $userService
) : self
{
    $userId = $metadata["userId"];
    if (!$userService->isAdmin($userId)) {
        throw new \InvalidArgumentException("You need to be administrator in order to register new product");
    }

    return new self($command->getProductId(), $command->getCost(), $userId);
}
```

Great, there is no way to bypass the constraint now. The **isAdmin constraint** must be satisfied in order to register new product. \
\
Let's correct our testing class. &#x20;

```php
public function run() : void
{
    $this->commandBus->sendWithRouting(
        "product.register",
        \json_encode(["productId" => 1, "cost" => 100]),
        "application/json",
        [
            "userId" => 1
        ]
    );

    $this->commandBus->sendWithRouting(
        "product.changePrice",
        \json_encode(["productId" => 1, "cost" => 110]),
        "application/json",
        [
            "userId" => 1
        ]
    );

    echo $this->queryBus->sendWithRouting("product.getCost", \json_encode(["productId" => 1]), "application/json");
}
```

{% hint style="success" %}
Let's run our testing command:
{% endhint %}

```php
bin/console ecotone:quickstart
Running example...
Product with id 1 was registered!
110
Good job, scenario ran with success!
```

### Injecting arguments

**Ecotone** inject arguments based on **Parameter Converters**.\
Parameter converters , tells **Ecotone** how to resolve specific parameter and what kind of argument it is expecting.  The one used for injecting services like **UserService** is **Reference** parameter converter.\
Let's see how could we use it in our **product.register** command handler.&#x20;

Let's suppose UserService is registered under **user-service** in Dependency Container. Then we would need to set up the `CommandHandler`like below.

```php
#[CommandHandler("product.register")]
public static function register(
    RegisterProductCommand $command, 
    array $metadata, 
    #[Reference("user-service")] UserService $userService
) : self
```

`Reference`- Does inject service from Dependency Container. If **referenceName**`,` which is name of the service in the container is not given, then it will take the class name as default.

`Payload` - Does inject payload of the [message](../messaging/messaging-concepts/message.md). In our case it will be the command itself

`Headers` - Does inject all headers as array.

`Header` - Does inject single header from the [message](../messaging/messaging-concepts/message.md).  \
\
There is more to be said about this, but at this very moment, it will be enough for us to know that such possibility exists in order to continue.\
You may read more detailed description in [Method Invocation section.](../messaging/conversion/method-invocation.md) \


### Default Converters

**Ecotone**, if parameter converters are not passed provides default converters. \
First parameter is always **Payload**`.` \
The second parameter, if is **array** then **Headers** converter is taken, otherwise if class type hint is provided for parameter, then **Reference** converter is picked. \
\
If we would want to manually configure parameters for **product.register** Command Handler, then it would look like this:

```php
#[CommandHandler("product.register")]
public static function register(
    #[Payload] RegisterProductCommand $command, 
    #[Headers] array $metadata, 
    #[Reference] UserService $userService
) : self
{
    // ...
}
```

We could also inject specific header and let Ecotone convert it directly to specific object (if we have Converter registered):

```php
#[CommandHandler("product.register")]
public static function register(
    #[Payload] RegisterProductCommand $command, 
    // injecting specific header and doing the conversion string to UserId
    #[Header("userId")] UserId $metadata, 
    #[Reference] UserService $userService
) : self
{
    // ...
}
```

{% hint style="success" %}
Great, we have just finished Lesson 4!

In this Lesson we learned about using Metadata to provide extra information to our Message.\
Besides we took a look on how arguments are injected into endpoint and how we can make use of it.\
\
Now we will learn about powerful Interceptors, which can be describes as Middlewares on steroids.
{% endhint %}
