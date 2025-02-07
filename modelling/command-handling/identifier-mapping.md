# Identifier Mapping

When loading Aggregates or Sagas we need to know what Identifier should be used for that. \
This depending on the business feature we work may require different approaches. \
In this section we will dive into different solutions which we can use.

## Auto-Mapping from the Command/Event

Ecotone resolves the mapping automatically, when Identifier in the Aggregate/Saga is named the same as the property in the Command/Event.

```php
 #[Aggregate]
class Product
{
    #[Identifier]
    private string $productId;
```

then, if Message has **productId**, it will be used for Mapping:

```php
class ChangePriceCommand
{
    private string $productId;
    private Money $priceAmount;
}
```

{% hint style="success" %}
You may use multiple aggregate identifiers or identifier as objects (e.g. Uuid) as long as they provide `__toString` method
{% endhint %}

## Expose Identifier using Method

We may also expose identifier over public method by annotating it with attribute **IdentifierMethod("productId").**

```php
#[Aggregate]
class Product
{
    private string $id;
    
    #[IdentifierMethod("productId")]
    public function getProductId(): string
    {
        return $this->id;
    }
```

## Targeting Identifier from Event/Command

If the property name is different than Identifier in the Aggregate/Saga, we need to give `Ecotone` a hint, how to correlate identifiers. \
We can do that using TargetIdentifier attribute, which states to which Identifier given property references too:

```php
class SomeEvent
{
    #[TargetIdentifier("orderId")] 
    private string $purchaseId;
}
```

## Targeting Identifier from Metadata

When there is no property to correlate inside **Command** or **Event**, we can use Identifier from Metadata.\
When we've the identifier inside `Metadata` then we can use **identifierMetadataMapping**`.`\
\
Suppose the **orderId** identifier is available in metadata under key **orderNumber**, then we can then use this mapping:

```php
#[EventHandler(identifierMetadataMapping: ["orderId" => "orderNumber"])]
public function failPayment(PaymentWasFailedEvent $event, CommandBus $commandBus) : self 
{
   // do something with $event
}
```

{% hint style="success" %}
We can make use of `Before` or `Presend` [Interceptors](../extending-messaging-middlewares/interceptors/) to enrich event's metadata with required identifiers.
{% endhint %}

## Dynamic Identifier

We may provide Identifier dynamically using Command Bus. This way we can state explicitly what Aggregate/Saga instance we do refer too. Thanks to we don't need to define Identifier inside the Command and we can skip any kind of mapping.

In some scenario we won't be in deal to create an Command class at all. For example we may provide block user action, which changes the status:

```php
$this->commandBus->sendWithRouting('user.block', metadata:
    'aggregate.id' => $userId // This way we provide dynamic identifier
])
```

```php
#[CommandHandler('user.block')]
public function block() : void
{
    $this->status = 'blocked';
}
```

{% hint style="success" %}
Event so we are using "aggregate.id" in the metadata, this will work exactly the same for Sagas. Therefore if we want to trigger Message Handler on the Saga, we can use "aggregate.id" too.
{% endhint %}

## Advanced Identifier Mapping

There may be cases where more advanced mapping may be needed. In those cases we can use identifier mapping based on [Expression Language](https://symfony.com/doc/current/components/expression_language.html).

When using **identifierMapping** configuration, we get access to the Message fully and to Dependency Container. To access specific part we will be using:

* **payload** **->** Represents our Event/Command class
* **headers ->** Represents our Message's metadata
* **reference('name') ->** Allow to access given service from our Dependency Container

1. Suppose the `orderId` identifier is available in metadata under key `orderNumber`, then we can tell Message Handler to use this mapping:

```php
#[EventHandler(identifierMapping: ["orderId" => "headers['orderNumber']"])]
public function failPayment(PaymentWasFailedEvent $event, CommandBus $commandBus) : void 
{
   // do something with $event
}
```

2. Suppose our Identifier is an Email object within Command class and we would like to normalize before it's used for fetching the Aggregate/Saga:

```php
class BlockUser
{
    private Email $email;
    
    (...)
    
    public function getEmail(): Email
    {
        return $this->email;
    }
}
```

```php
#[CommandHandler(identifierMapping: [
   "email" => "payload.getEmail().normalize()"]
)]
public function block(BlockUser $command) : void
{
   // do something with $command
}
```

3. Suppose we receive external order id, however we do have in database our internal order id that should be used as Identifier. We could then have a Service registered in DI under **"orderIdExchange":**

```php
class OrderIdExchange
{
    public function exchange(string $externalOrderId): string
    {
        // do the mapping
        
        return $internalOrderId;
    }
}
```

Then we can make use of it in our identifier Mapping

```php
#[EventHandler(identifierMapping: [
   "orderId" => "reference('orderIdExchange').exchange(payload.externalOrderId())"
])]
public function when(OrderCancelled $event) : void
{
   // do something with $event
}
```

