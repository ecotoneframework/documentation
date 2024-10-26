# Asynchronous Message Bus (Gateways)

We may extend Gateways functionality with asynchronocity. This way we can pass any Message via given Message Channel first.

{% hint style="success" %}
Asynchronous Gateways are available as part of **Ecotone Enterprise.**
{% endhint %}

## Asynchronous Gateways

To make Gateway Asynchronous we will use Asynchronous Attribute, just like with Asynchronous Message Handlers. We may can extend any types of Gateways: Command/Event/Query Buses, [Business Interfaces](../command-handling/business-interface/) or custom [Gateways](../../messaging/messaging-concepts/messaging-gateway.md).

To build for example a **CommandBus** which will send Messages over **async** channel, we will simply [extend a CommandBus interface](../extending-messaging-middlewares/extending-message-buses-gateways.md), and add our method.

```php
#[Asynchronous("async")]
interface AsynchronousCommandBus extends CommandBus
{

}
```

then we all Commands that will be triggered via **AsynchronousCommandBus** will go over async channel.&#x20;

{% hint style="success" %}
It's enough to extend given **CommandBus** with custom interface to register new abstraction in  Gateway in Dependency Container.&#x20;
{% endhint %}

Having **asynchronous CommandBus** is especially useful, if given Message Handler is not meant be executed asynchronous by default.

```php
#[CommandHandler]
public function placeOrder(PlaceOrderCommand $command) : void
{
   // do something with $command
}
```

then when using standard CommandBus above Command Handler will be executed synchronous, when using AsynchronousCommandBus it will be done asynchronously.

## Outbox Event Bus

It's easy to build an Outbox pattern using this Asynchronous Gateways. Just make use of[ Dbal Message Channel](../../modules/dbal-support.md) to push Messages over Database Channel.&#x20;

```php
#[Asynchronous("outbox")]
interface OutboxEventBus extends EventBus {}
```

and then register dbal channel

```php
#[ServiceContext] 
public function orderChannel()
{
    return DbalBackedMessageChannelBuilder::create("orders");
}
```

Then whenever we will send Events within Command Handler (which is wrapped in transaction by default while using Dbal Module). Messages will be commited as part of same transaction.

```php
#[CommandHandler]
public function placeOrder(PlaceOrderCommand $command, OutboxEventBus $eventBus) : void
{
   // do something with $command
   
   $eventBus->publish(new OrderWasPlaced());
}
```
