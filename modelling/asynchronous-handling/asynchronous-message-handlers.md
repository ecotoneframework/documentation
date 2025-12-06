# Asynchronous Message Handlers

## Running Asynchronously

**Ecotone** does allow for easy change from synchronous to asynchronous execution of given **Message Handler**.

In order to run Command Handler asynchronously we need to mark it as **Asynchronous**.

```php
#[Asynchronous("orders")]
#[CommandHandler(endpointId: "place_order_endpoint")
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

## Message Channel

The asynchronous attribute states what Channel reference we want to use:

```php
#[Asynchronous("orders")]
```

**The "orders" string is the name of our Message Channel.** We use this name to reference which implementation we want to use—whether it's an in-memory channel for testing, a database queue, or RabbitMQ. This naming approach keeps our business code clean and independent from infrastructure choices.

To configure a specific implementation like a database channel, we use a [**ServiceContext**](../../messaging/service-application-configuration.md) class.&#x20;

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

That's all the configuration we need! Now whenever we reference "orders" in our handler attributes, Ecotone automatically uses this database channel. Our handlers stay exactly the same whether we're using in-memory channels for testing or database channels for production—the only difference is this single configuration change.&#x20;

## Running Message Consumer

We can first list all of the Message Consumers we have available for running:

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

Then in order to run our Message Consumer, we will use **ecotone:run** console command:

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

## Dynamic Configuration

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

## Available Providers (Types)

There are multiple different implementation which we can use:

* [AMQP Support (RabbitMQ)](../../modules/amqp-support-rabbitmq/#message-channel)
* [Kafka Support](../../modules/kafka-support/)
* [DBAL Support](../../modules/dbal-support.md#message-channel)
* [SQS Support](../../modules/amazon-sqs-support.md)
* [Redis Support](../../modules/redis-support.md)
* [Symfony Messenger Transport Support](../../modules/symfony/symfony-messenger-transport.md)
* [Laravel Queues](../../modules/laravel/laravel-queues.md)
* [In Memory Channels](../testing-support/testing-asynchronous-messaging.md)

## Multiple Asynchronous Endpoints

Using single asynchronous channel we may register multiple endpoints.\
This allow for registering single asynchronous channel for whole Aggregate or group of related Command/Event Handlers.

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

All asynchronous endpoints are marked with special attribute`Ecotone\Messaging\Attribute\AsynchronousRunningEndpoint`\
If you want to [intercept](../extending-messaging-middlewares/interceptors/) all polling endpoints you should make use of [annotation related point cut](../extending-messaging-middlewares/interceptors/#pointcut) on this.

## Endpoint Id

Each Asynchronous Message Handler requires us to define **"endpointId"**. It's unique identifier of your Message Handler.

```php
#[Asynchronous("orders")]
#[EventHandler(endpointId: "order_was_placed") // Your important endpoint Id
public function when(OrderWasPlaced $event) : void {}
```

The Endpoint ID travels with your message as part of the headers to your message channel. Once we consume the message from the Message Channel, Ecotone uses this ID to route it to the correct Message Handler. This completely decouples our messages from specific handler classes and methods—we can refactor, rename, or move our handlers around without breaking message routing.
