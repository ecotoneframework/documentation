---
description: PHP Conversion
---

# Lesson 3: Converters

{% hint style="info" %}
Not having code for _Lesson 3?_ \
\
`git checkout lesson-3`
{% endhint %}

### Conversion

Command, queries and events are not always objects. When they travel via different asynchronous channels, they are converted to simplified format, like `JSON` or `XML`.  
At the level of application however we want to deal with it in `PHP` format, as objects or arrays.

Moving from one format to another requires conversion. `Ecotone` does provide extension point in which we can integrate different [Media Type](https://pl.wikipedia.org/wiki/Typ\_MIME) converters.&#x20;

Let's build our first converter from `JSON` to our `PHP` format. In order to do that, we will need to implement `Converter` interface and mark it with `MediaTypeConverter().`

```php
<?php

namespace App\Domain\Product;

use Ecotone\Messaging\Attribute\MediaTypeConverter;
use Ecotone\Messaging\Conversion\Converter;
use Ecotone\Messaging\Conversion\MediaType;
use Ecotone\Messaging\Handler\TypeDescriptor;

#[MediaTypeConverter]
class JsonToPHPConverter implements Converter
{
    public function matches(TypeDescriptor $sourceType, MediaType $sourceMediaType, TypeDescriptor $targetType, MediaType $targetMediaType): bool
    {

    }

    public function convert($source, TypeDescriptor $sourceType, MediaType $sourceMediaType, TypeDescriptor $targetType, MediaType $targetMediaType)
    {

    }
}
```

1. `TypeDescriptor` - Describes type in PHP format. This can be `class, scalar (int, string), array` etc.&#x20;
2. `MediaType` - Describes Media type format. This can be `application/json`, `application/xml` etc.&#x20;
3. `$source` - is the actual data to be converted.&#x20;

Let's start with implementing `matches` method. Which tells us, if this converter can do conversion from one type to another.

```php
public function matches(TypeDescriptor $sourceType, MediaType $sourceMediaType, TypeDescriptor $targetType, MediaType $targetMediaType): bool
{
    return $sourceMediaType->isCompatibleWith(MediaType::createApplicationJson()) // if source media type is JSON
        && $targetMediaType->isCompatibleWith(MediaType::createApplicationXPHP()); // and target media type is PHP
}
```

This will tell `Ecotone` that in case source media type is `JSON` and target media type is `PHP`, then it should use this converter. \
Now we need to implement the convert method now. We will do it with pretty naive solution, just to proof the concept.&#x20;

```php
public function convert($source, TypeDescriptor $sourceType, MediaType $sourceMediaType, TypeDescriptor $targetType, MediaType $targetMediaType)
{
    $data = \json_decode($source, true, 512, JSON_THROW_ON_ERROR);
    // $targetType hold the class, which we will convert to
    switch ($targetType->getTypeHint()) {
        case RegisterProductCommand::class: {
            return RegisterProductCommand::fromArray($data);
        }
        case GetProductPriceQuery::class: {
            return GetProductPriceQuery::fromArray($data);
        }
        default: {
            throw new \InvalidArgumentException("Unknown conversion type");
        }
    }
}
```

{% hint style="info" %}
Normally you would inject into Converter class, some kind of serializer used within your application for example `JMS Serializer` or `Symfony Serializer` to make the conversion.
{% endhint %}

And let's add `fromArray` method to `RegisterProductCommand` and `GetProductPriceQuery.`

```php
class GetProductPriceQuery
{
    private int $productId;

    public function __construct(int $productId)
    {
        $this->productId = $productId;
    }

    public static function fromArray(array $data) : self
    {
        return new self($data['productId']);
    }
```

```php
class RegisterProductCommand
{
    private int $productId;

    private int $cost;

    public function __construct(int $productId, int $cost)
    {
        $this->productId = $productId;
        $this->cost = $cost;
    }

    public static function fromArray(array $data) : self
    {
        return new self($data['productId'], $data['cost']);
    }
```

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

If we call our testing command now, everything is going fine, but we still send `PHP objects` instead of `JSON`, right?\
In order to start sending `commands` and `queries` in different format, we need to provide our handlers with routing key. So `Command and Query buses` will know, where to route the message.&#x20;

{% hint style="info" %}
You may think now of routing key, as a message name used in different CQRS frameworks in order to route the message to specific handler. This is yet much more powerful, and we will see why in next lessons.&#x20;
{% endhint %}

```php
#[CommandHandler("product.register")]
public static function register(RegisterProductCommand $command) : self
{
    return new self($command->getProductId(), $command->getCost());
}

#[QueryHandler("product.getCost")] 
public function getCost(GetProductPriceQuery $query) : int
{
    return $this->cost;
}
```

Let's change our Testing class, so we call buses with `JSON` format.

```php
(...)

public function run() : void
{
    $this->commandBus->sendWithRouting("product.register", \json_encode(["productId" => 1, "cost" => 100]), "application/json");

    echo $this->queryBus->sendWithRouting("product.getCost", \json_encode(["productId" => 1]), "application/json");
}
```

We make use of different method now `convertAndSend.` \
It takes as first argument `routing key` to which we want to send the message. \
The second argument describes the `format` of message we send.\
Third is the data to send itself, in this case command formatted as `JSON`.

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

### Ecotone JMS Converter

`Ecotone` comes with integration with [JMS Serializer](https://jmsyst.com/libs/serializer) and extending it with extra features.\
Let's replace our own written Converter with JMS Serializer integration. \
Let's download the Converter using [Composer](https://getcomposer.org). \
`composer require ecotone/jms-converter`

Let's remove `__construct` and`fromArray` methods from `RegisterProductCommand` `GetProductPriceQuery` and the `App\Domain\Product\JsonToPHPConverter` class completely, as we won't need it anymore.

{% hint style="success" %}
JMS creates cache to speed up serialization process. In case of problems with running this test command, try to remove your cache.\
Let's run our testing command:
{% endhint %}

```php
bin/console ecotone:quickstart
Running example...
Product with id 1 was registered!
100
Good job, scenario ran with success!
```

Do you wonder, how come, that we just deserialized our Command and Query classes without any additional code?\
Well we already described those classes using type hints or docblocks. \
`Ecotone JMS` reads properties and deserializes according to type hint or docblock if it is array.&#x20;

```php
Conversion Table examples:

private int $productId => int

private string $data => string

private \stdClass $data => \stdClass
 
/**
* @var \stdClass[] 
*/
private array $data => array<\stdClass>
```



Let's imagine we found out, that we have bug in our software. Our system users have registered product with negative price, which in result lowered the bill.&#x20;

`Product should be registered only with positive cost`\
\
We could put constraint in `Product`, validating the `Cost` amount. But this would assure us only in that place, that this constraint is met and we want to be sure, that the `Cost` is correct, whenever we make use of it, so we can avoid potential future bugs. \
To achieve that we will create _Value Object named `Cost`_that will handle the validation, during the construction.&#x20;

```php
namespace App\Domain\Product;

class Cost
{
    private int $amount;

    public function __construct(int $amount)
    {
        if ($amount <= 0) {
            throw new \InvalidArgumentException("The cost cannot be negative or zero, {$amount} given.");
        }
        
        $this->amount = $amount;
    }

    public function getAmount() : int
    {
        return $this->amount;
    }
    
    public function __toString()
    {
        return (string)$this->amount;
    }
}
```

Great, but where to convert the integer to the Cost class? We really don't want to burden our business logic with conversions. `Ecotone JMS` does provide extension points, so we can tell him, how to convert specific classes.&#x20;

{% hint style="info" %}
Normally you will like to delegate conversion to Converters, as we want to get our domain classes converted as fast as we can. The business logic should stay clean, so it can focus on the domain problems, not technical problems.
{% endhint %}

Let's create class `App\Infrastructure\Converter\CostConverter.` We will put it in different namespace, to separate it from the domain.

```php
namespace App\Infrastructure\Converter;

use App\Domain\Product\Cost;
use Ecotone\Messaging\Attribute\Converter;

class CostConverter
{
    #[Converter]
    public function convertFrom(Cost $cost) : int
    {
        return $cost->getAmount();
    }

    #[Converter]
    public function convertTo(int $amount) : Cost
    {
        return new Cost($amount);
    }
}
```

We mark the methods with`Converter()`, so `Ecotone` can read parameter type and return type in order to know, how he can convert from scalar/array to specific class and vice versa.\
\
Let's change our command and aggregate class, so it can use the Cost directly.

```php
class RegisterProductCommand
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

The `$cost` class property will be automatically converted from `integer` to `Cost`after calling `Command Bus`.

```php
class Product
{
    use WithAggregateEvents;

    #[AggregateIdentifier]
    private int $productId;

    private Cost $cost;

    private function __construct(int $productId, Cost $cost)
    {
        $this->productId = $productId;
        $this->cost = $cost;

        $this->recordThat(new ProductWasRegisteredEvent($productId));
    }

    #[CommandHandler("product.register")]
    public static function register(RegisterProductCommand $command) : self
    {
        return new self($command->getProductId(), $command->getCost());
    }

    #[QueryHandler("product.getCost")]
    public function getCost(GetProductPriceQuery $query) : Cost
    {
        return $this->cost;
    }
}
```

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

{% hint style="info" %}
We have used Converter in order to construct the Cost object using constructor from string, to validate correctness of the incoming data. \
However you may find Native Conversion enough in your case without the need to create conversion methods.\
To get more information, read [Native Conversion](../modules/jms-converter.md#native-conversion)
{% endhint %}

{% hint style="success" %}
The command which we send from outside (to the Command Bus) is still the same, as before. \
We changed the internals of the domain, without affecting consumers of our API. \
In this Lesson we learned how to make use of Converters.\
\
Great, we just finished Lesson 3! \
\
In next Lesson we will learn and Method Invocation and Metadata
{% endhint %}
