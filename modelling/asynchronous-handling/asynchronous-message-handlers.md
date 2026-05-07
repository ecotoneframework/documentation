---
description: Making message handlers asynchronous with a single attribute
---

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

### Asynchronous Execution Attributes (Enterprise)

{% hint style="success" %}
This is Enterprise feature. To use it, **email us at** "**support@simplycodedsoftware.com**" **to receive trial key**. **Production license keys** are available at [https://ecotone.tech](https://ecotone.tech/pricing).
{% endhint %}

When database transactions are globally enabled for a message channel, all async handlers on that channel are wrapped in a transaction. However, some handlers may not need a transaction — for example, a handler that only calls a 3rd party API, sends an email, or triggers a webhook. Wrapping such handlers in an unnecessary database transaction wastes resources and holds connections open longer than needed.

With **`asynchronousExecution`** attributes on the `#[Asynchronous]` attribute, you can configure runtime behavior per-handler — applied when the polling consumer processes the Message, not at the synchronous bus call. Attributes must implement the `AsynchronousEndpointAttribute` interface.

```php
use Ecotone\Messaging\Attribute\Asynchronous;
use Ecotone\Messaging\Attribute\WithoutDatabaseTransaction;

#[Asynchronous("orders", asynchronousExecution: [new WithoutDatabaseTransaction()])]
#[CommandHandler(endpointId: "send_order_confirmation")]
public function sendConfirmation(SendOrderConfirmation $command, EmailService $emailService): void
{
    // This handler only sends an email — no database work needed.
    // The global transaction is skipped for this specific handler,
    // while other handlers on the "orders" channel still use transactions.
    $emailService->send($command->email, $command->orderId);
}
```

The attributes are resolved at runtime when the message is consumed, and are available to interceptors targeting `AsynchronousRunningEndpoint`.

#### Built-in Asynchronous Execution Attributes

| Attribute | Effect |
|---|---|
| `WithoutDatabaseTransaction` | Skips the global DBAL transaction interceptor for this handler |
| `WithoutMessageCollector` | Skips message collection — events are sent directly to channels during handler execution instead of being buffered and released after completion |
| `ErrorChannel` | Routes failures from this handler to a per-handler Error Channel — see [Per-Handler Error Channel](../recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/#per-handler-error-channel-for-asynchronous-handlers) |
| `DelayedRetry` | Retries the handler with the configured delay/backoff, then routes to a dead letter channel on exhaustion — see [Per-Handler Delayed Retry](../recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/#per-handler-delayed-retry-for-asynchronous-handlers). Mutually exclusive with `ErrorChannel`. |

#### Combining Asynchronous Execution Attributes

Multiple attributes compose on a single handler. A common combination is `WithoutDatabaseTransaction` together with `ErrorChannel`: the handler doesn't need a DB transaction because it only calls a 3rd party API, but you still want failures captured to a dedicated error channel for retry or review.

```php
use Ecotone\Messaging\Attribute\Asynchronous;
use Ecotone\Messaging\Attribute\ErrorChannel;
use Ecotone\Messaging\Attribute\WithoutDatabaseTransaction;

#[Asynchronous(
    "orders",
    asynchronousExecution: [
        new WithoutDatabaseTransaction(),
        new ErrorChannel("emailDeliveryErrors"),
    ]
)]
#[CommandHandler(endpointId: "send_order_confirmation")]
public function sendConfirmation(SendOrderConfirmation $command, EmailService $emailService): void
{
    // No DB transaction is opened around this handler — the global transaction interceptor is skipped.
    // If the email API throws, the failed Message is captured to "emailDeliveryErrors" instead of
    // propagating up the polling consumer; other handlers on the "orders" channel keep their defaults.
    $emailService->send($command->email, $command->orderId);
}
```

The attributes are independent — each one is read by the interceptor it concerns (`WithoutDatabaseTransaction` by the DBAL transaction interceptor, `ErrorChannel` by the polling consumer's error interceptor). Order in the array does not matter.

#### Custom Asynchronous Execution Attributes

You can create your own attributes and inject them into interceptors:

```php
use Ecotone\Messaging\Attribute\AsynchronousEndpointAttribute;

#[Attribute]
class CustomDelayedRetry implements AsynchronousEndpointAttribute
{
    public function __construct(public int $maxRetries = 3) {}
}
```

```php
#[Asynchronous("orders", asynchronousExecution: [new CustomDelayedRetry(maxRetries: 5)])]
#[CommandHandler(endpointId: "place_order_endpoint")]
public function placeOrder(PlaceOrderCommand $command): void {}
```

Then access it in an interceptor:

```php
#[Around(pointcut: AsynchronousRunningEndpoint::class)]
public function retry(
    MethodInvocation $invocation,
    ?CustomDelayedRetry $policy = null
): mixed {
    // $policy is injected from the handler's asynchronousExecution attributes
    return $invocation->proceed();
}
```

## Endpoint Id

Each Asynchronous Message Handler requires us to define **"endpointId"**. It's unique identifier of your Message Handler.

```php
#[Asynchronous("orders")]
#[EventHandler(endpointId: "order_was_placed") // Your important endpoint Id
public function when(OrderWasPlaced $event) : void {}
```

The Endpoint ID travels with your message as part of the headers to your message channel. Once we consume the message from the Message Channel, Ecotone uses this ID to route it to the correct Message Handler. This completely decouples our messages from specific handler classes and methods—we can refactor, rename, or move our handlers around without breaking message routing.
