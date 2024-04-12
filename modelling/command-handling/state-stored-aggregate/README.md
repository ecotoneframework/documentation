---
description: DDD Aggregates PHP
---

# Aggregate Introduction

This chapter will cover the basics on how to implement an [Aggregate](../../modelling-1.md#aggregates). \
We will be using Command Handlers in this section, so ensure reading [External Command Handler](../external-command-handlers/) section first, to understand how Command are sent and handled.

## Aggregate Command Handlers

Working with Aggregate Command Handlers is the same as with [External Command Handlers](../external-command-handlers/).\
We mark given method with `Command Handler` attribute and Ecotone will register it as Command Handler.

In most common scenarios, Command Handlers are used as boilerplate code, which fetch the aggregate, execute it and then save it.

```php
$product = $this->repository->getById($command->id());
$product->changePrice($command->getPriceAmount());
$this->repository->save($product);
```

This is non-business code that is often duplicated wit each of the Command Handler we introduce. \
Ecotone wants to shift the developer focus on the business part of the system, this is why this is abstracted away in form of Aggregate.

```php
 #[Aggregate]
class Product
{
    #[Identifier]
    private string $productId;
```

By providing `Identifier` attribute on top of property in your Aggregate, we state that this is identifier of this Aggregate (Entity in Symfony/Doctrine world, Model in Laravel world). \
This is then used by Ecotone to fetch your aggregate automatically.

However Aggregates need to be [fetched from repository](../repository.md) in order to be executed. \
When we will send an Command, Ecotone will use property with same name from the Command instance to fetch the Aggregate.

```php
class ChangePriceCommand
{
    private string $productId; // same property name as Aggregate's Identifier
    private Money $priceAmount;
```

{% hint style="success" %}
You may read more about Identifier Mapping and more advanced scenarios  in [related section](../identifier-mapping.md).
{% endhint %}

When identifier is resolved, Ecotone use `repository` to fetch the aggregate and then call the method and then save it. So basically do all the boilerplate for you.

{% hint style="success" %}
To implement repository reference to [this section](../repository.md).\
You may use inbuilt repositories, so you don't need to implement your own.\
Ecotone provides [`Event Sourcing Repository`](../../event-sourcing/), [`Document Store Repository`](../../../messaging/document-store.md#storing-aggregates-in-your-document-store), integration with [Doctrine ORM](../../../modules/symfony/doctrine-orm.md) or [Eloquent](../../../modules/laravel/eloquent.md).
{% endhint %}

## State-Stored Aggregate

An Aggregate is a regular object, which contains state and methods to alter that state. It can be described as Entity, which carry set of behaviours. \
When creating the Aggregate object, you are creating the _Aggregate Root_.&#x20;

```php
 #[Aggregate] // 1
class Product
{
    #[Identifier] // 2
    private string $productId;

    private string $name;

    private integer $priceAmount;
    
    private function __construct(string $orderId, string $name, int $priceAmount)
    {
        $this->productId = $orderId;
        $this->name = $name;
        $this->priceAmount = $priceAmount;
    }

    #[CommandHandler]  //3
    public static function register(RegisterProductCommand $command) : self
    {
        return new self(
            $command->getProductId(),
            $command->getName(),
            $command->getPriceAmount()
        );
    }
    
    #[CommandHandler] // 4
    public function changePrice(ChangePriceCommand $command) : void
    {
        $this->priceAmount = $command->getPriceAmount();
    }
}
```

1. `Aggregate` tells Ecotone, that this class should be registered as Aggregate Root.
2.  `Identifier` is the external reference point to Aggregate.&#x20;

    This field tells Ecotone to which Aggregate a given Command is targeted.
3. `CommandHandler` defined on static method acts as _factory method_. Given command it should return _new instance_ of specific aggregate, in that case new Product.
4. `CommandHandler` defined on non static class method is place where you would make changes to existing aggregate, fetched from repository.
