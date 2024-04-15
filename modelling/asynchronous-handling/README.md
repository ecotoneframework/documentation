---
description: Asynchronous PHP
---

# Asynchronous Handling and Scheduling

## Running Asynchronously

`Ecotone` does allow for easy change from synchronous to asynchronous execution of given `Message Handler`.

In order to run Command Handler asynchronously we need to mark it as `Asynchronous`.

```php
#[Asynchronous("orders")]
#[CommandHandler("order.place", "place_order_endpoint")
public function placeOrder(PlaceOrderCommand $command) : void
{
   // do something with $command
}
```

The same way we define for Event Handlers:

```php
#[Asynchronous("orders")]
#[EventHandler(endpointId: "order_was_placed")
public function when(OrderWasPlaced $event) : void
{
   // do something with $event
}
```

We need to add **endpointId** on our endpoint's annotation, this will be used to route the Message in isolation to our **Message Handlers**.

## Message Channel Name

```php
#[Asynchronous("orders")]
```

The "orders" string is actually a name of our Message Channel. This way we reference to specific implementation which we would like to use. To provide specific implementation like for example Database Channel, we would use **ServiceContext**.

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function databaseChannel()
    {
        return DbalBackedMessageChannelBuilder::create('orders');
    }
}
```

This is basically all we need to configure. Now database channel called **orders** will be used, whenever we will use Attribute with this name.

There are multiple different implementation which we can use:

## Available Asynchronous Message Channels

At this moment following modules with pollable channels are available:

* [AMQP Support (RabbitMQ)](../../modules/amqp-support-rabbitmq.md#message-channel)
* [DBAL Support](../../modules/dbal-support.md#message-channel)
* [SQS Support](../../modules/amazon-sqs-support.md)
* [Redis Support](../../modules/redis-support.md)
* [Symfony Messenger Transport Support](../../modules/symfony/symfony-messenger-transport.md)
* [Laravel Queues](../../modules/laravel/laravel-queues.md)

{% hint style="success" %}
If you're choose [Dbal Message Channel](../../modules/dbal-support.md#message-channel) `Ecotone` will use [Outbox Pattern](../../quick-start-php-ddd-cqrs-event-sourcing/resiliency-and-error-handling.md) to atomically store your changes and published messages.
{% endhint %}

{% hint style="info" %}
Currently available Message Channels are integrated via great library [enqueue](https://github.com/php-enqueue/enqueue).
{% endhint %}

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:list
+--------------------+
| Endpoint Names     |
+--------------------+
| orders             |
+--------------------+
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:list
+--------------------+
| Endpoint Names     |
+--------------------+
| orders             |
+--------------------+
```
{% endtab %}

{% tab title="Lite" %}
```
$consumers = $messagingSystem->list()
```
{% endtab %}
{% endtabs %}

After setting up Pollable Channel we can run the endpoint:

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:run orders -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:run orders -vvv
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->run("orders");
```
{% endtab %}
{% endtabs %}

## Running Asychronous Endpoint (PollingMetadata)

### Dynamic Configuration

You may set up running configuration for given consumer while running it.

* `handledMessageLimit` - Amount of messages to be handled before stopping consumer
* `executionTimeLimit` - How long consumer should run before stopping (milliseconds)
* `finishWhenNoMessages` - Consumers will be running as long as there will be messages to consume
* `memoryLimit` - How much memory can be consumed by before stopping consumer (Megabytes)
* `stopOnFailure` - Stop consumer in case of exception

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:run orders 
    --handledMessageLimit=5 
    --executionTimeLimit=1000 
    --finishWhenNoMessages
    --memoryLimit=512
    --stopOnFailure
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:run orders
    --handledMessageLimit=5 
    --executionTimeLimit=1000
    --finishWhenNoMessages 
    --memoryLimit=512
    --stopOnFailure
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->run(
    "orders", 
    ExecutionPollingMetadata::createWithDefault()
        ->withHandledMessageLimit(5)
        ->withMemoryLimitInMegabytes(100)
        ->withExecutionTimeLimitInMilliseconds(1000)
        ->withFinishWhenNoMessages(true)
        ->withStopOnError(true)
);
```
{% endtab %}
{% endtabs %}

### Static Configuration

Using [Service Context ](../../messaging/service-application-configuration.md)configuration for statically configuration.

```php
class Configuration
{    
    #[ServiceContext]
    public function configuration() : array
    {
        return [
            PollingMetadata::create("orders")
                 ->setErrorChannelName("errorChannel")
                 ->setInitialDelayInMilliseconds(100)
                 ->setMemoryLimitInMegaBytes(100)
                 ->setHandledMessageLimit(10)
                 ->setExecutionTimeLimitInMilliseconds(100)
                 ->withFinishWhenNoMessages(true)
        ];
    }
}
```

{% hint style="info" %}
Dynamic configuration overrides static
{% endhint %}

## Multiple Asynchronous Endpoints

Using single asynchronous channel we may register multiple endpoints. \
This allow for registering single asynchronous channel for whole Aggregate or group of related Command/Event Handlers.&#x20;

```php
#[Asynchronous("orders")]
#[EventHandler]
public function onSuccess(SuccessEvent $event) : void
{
}

#[Asynchronous("orders")]
#[EventHandler]
public function onSuccess(FailureEvent $event) : void
{
}
```

#### Asynchronous Class

You may put `Asynchronous` on the class, level so all the endpoints within a class will becomes asynchronous.

## Intercepting asynchronous endpoint

All asynchronous endpoints are marked with special attribute`Ecotone\Messaging\Attribute\AsynchronousRunningEndpoint` \
If you want to [intercept](../extending-messaging-middlewares/interceptors.md) all polling endpoints you should make use of [annotation related point cut](../extending-messaging-middlewares/interceptors.md#pointcut) on this.

## Endpoint Id

Each Asynchronous Message Handler requires us to define **"endpointId"**.  It's unique identifier of your Message Handler.

```php
#[Asynchronous("orders")]
#[EventHandler(endpointId: "order_was_placed") // Your important endpoint Id
public function when(OrderWasPlaced $event) : void {}
```

Endpoint Id goes in form of Headers to your Message Queue. After Message is consumed from the Queue, Message will be directed to your Message Handler having given endpoint Id. \
This decouples the Message completely from the Message Handler Class and Method and Command/Event Class.&#x20;

{% hint style="success" %}
EndpointId ensures we can freely refactor our code and it will be backward compatible with Messages in the Queues. This means we can move the method and class to different namespaces, change the Command class names and as long as **endpointId** is kept the same Message will be delivered correctly.&#x20;
{% endhint %}

## Materials

### Demo implementation

You may find demo implementation [here](https://github.com/ecotoneframework/quickstart-examples/tree/main/OutboxPattern).

### Links

* [Asynchronous PHP To Support Stability Of Your Application](https://blog.ecotone.tech/asynchronous-php/) \[Article]
* [Asynchronous Messaging in Laravel](https://blog.ecotone.tech/ddd-and-messaging-with-laravel-and-ecotone/) \[Article]
* [Testing Asynchronous Messaging](../testing-support/testing-asynchronous-messaging.md) \[Documentation]
* [Testing Asynchronous Message Driven Architecture](https://blog.ecotone.tech/testing-messaging-architecture-in-php/) \[Article]
