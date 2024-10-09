---
description: Symfony Messenger CQRS DDD
---

# Symfony Messenger Transport

## Symfony Messenger Transport Integration

Ecotone comes with Symfony Messenger Transport integration. \
We may use [Transports](https://symfony.com/doc/current/messenger.html#transport-configuration) as our [Message Channels](../../messaging/messaging-concepts/message-channel.md). \
This way we can work with familiar environment and reuse already set up `Symfony Serializer`.

### Asynchronous Message Handler

Suppose we have **"async"** transport:

```yaml
framework:
  messenger:
    transports:
      async: 'doctrine://default?queue_name=async'
```

Then we can register it using [Service Context](../../messaging/service-application-configuration.md) in Ecotone:

```php
final class MessagingConfiguration
{
    #[ServiceContext]
    public function asyncChannel()
    {
        return SymfonyMessengerMessageChannelBuilder::create("async");
    }
}
```

After that we can start using it as any other [asynchronous channel](../../modelling/asynchronous-handling/).

```php
#[Asynchronous('async')]
#[CommandHandler(endpointId:"place_order_endpoint")]
public function placeOrder(PlaceOrder $command): void
{
    // place the order
}
```

### Trigger Command/Event/Query via Ecotone Bus

In order to trigger `Command` `Event` or `Query` Handler, we will be sending the `Message` via given Ecotone's Bus, sending Messages via Symfony Bus will have no effect due to lack of information what kind of message it's.

{% hint style="info" %}
Ecotone provide Command/Event/Query buses, Symfony contains general Message Bus. \
It's important to distinguish between Message types as `Queries` are handled synchronously, `Commands` are point to point and target single Handler, `Events` on other hand are publish-subscribe which means multiple Handlers can subscribe to it.&#x20;
{% endhint %}

In order to trigger given Bus, inject CommandBus, EventBus or QueryBus (they are automatically available after installing Ecotone) and make use of them to send a Message.

```php
final class OrderController
{
    public function __construct(private CommandBus $commandBus) {}

    public function placeOrder(Request $request): Response
    {
        $command = $this->prepareCommand($request);

        $this->commandBus->send($command);

        return new Response();
    }
    
    (...)
```

### Running Message Consumer (Worker)

We will be running Message Consumer using "**ecotone:run"** command:

```bash
bin/console ecotone:run async -vvv
```

## Sending messages via routing

When sending command and events via routing, it's possible to use non-class types. In case of Symfony however, Messenger Transports require to Classes. To solve that Ecotone wraps simple types in a class and unwrap it on deserialization. Thanks to that we can Symfony Transport like any other Ecotone's Message Channel.

* Command Handler with command having `array payload`

```php
$this->commandBus->sendWithRouting('order.place', ["products" => [1,2,3]]);
```

```php
#[Asynchronous('async')]
#[CommandHandler('order.place', 'place_order_endpoint')]
public function placeOrder(array $payload): void
{
    // do something with 
}
```

* Command Handler inside [Aggregate](../../modelling/command-handling/state-stored-aggregate/) with command having `no payload` at all

```php
$this->commandBus->sendWithRouting('order.place', metadata: ["aggregate.id" => 123]);
```

```php
#[Aggregate]
class Order
{
    #[Asynchronous('async')]
    #[CommandHandler('cancel', 'cancel_order_endpoint')]
    public function cancel(): void
    {
        $this->isCancelled = true;
    }
(...)    
```

## Asynchronous Event Handlers

In case of sending events, we will be using **Event Bus**.

* EventBus is available out of the box in Dependency Container. Therefore all we need to do, is to inject it and publish an Event

```php
$this->eventBus->publish(new OrderWasPlaced($orderId));
```

* Subscribe to Event

```php
#[Asynchronous('async')]
#[EventHandler('order_was_placed_endpoint')]
public function notifyAbout(OrderWasPlaced $event): void
{
    // send notification
}
```
