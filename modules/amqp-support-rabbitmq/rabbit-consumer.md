# Rabbit Consumer

Set up production-grade RabbitMQ consumption with a single attribute. Built-in reconnection, graceful shutdown, and health checks replace custom consumer process management scripts.

**You'll know you need this when:**

* Your custom RabbitMQ consumer scripts need manual connection handling, reconnection logic, and shutdown management
* Consumer processes crash on connection drops and require external restart supervision
* You need health checks and graceful shutdown for containerized deployments
* You want to combine consumption with instant retry, error channels, and deduplication declaratively

{% hint style="success" %}
This feature is available as part of **Ecotone Enterprise.**
{% endhint %}

## Consume Messages from Queue

To consume Messages from Queue, it's enough to mark given method with AmqpConsumer attribute:

```php
#[RabbitConsumer(
    endpointId: 'orders_consumer',
    queueName: 'orders'
)]
public function handle(string $payload): void
{
    // handle
}
```

And then we can run related process using endpoint Id "**orders\_consumer**".\
Read more on running asynchronous processes in [related section](../../modelling/asynchronous-handling/asynchronous-message-handlers.md).

## Automatic Conversion

If we have our Custom Conversion or [JMS Module ](../jms-converter.md)installed, then we can leverage automatic conversion:

```php
#[RabbitConsumer(
    endpointId: 'orders_consumer',
    queueName: 'orders'
)]
public function handle(Order $payload): void
{
    // handle
}
```

Read more about Conversion in [related section](../../messaging/conversion/conversion/).

## Instant Retry&#x20;

In case of failure we may try to recover instantly. To do so we can provide **InstantRetry** attribute:

```php
#[InstantRetry(retryTimes: 3)]
#[RabbitConsumer(
    endpointId: 'orders_consumer',
    queueName: 'orders'
)]
public function handle(Order $payload): void
{
    // handle
}
```

Read more about Instant Retry in [related section](../../modelling/recovering-tracing-and-monitoring/resiliency/retries.md).

## Error Channel

To handle Message when failure happen, we may decide to send it to Error Channel. This can be used to for example store the Message for later review.&#x20;

```php
#[ErrorChannel('customErrorChannel')]
#[RabbitConsumer(
    endpointId: 'orders_consumer',
    queueName: 'orders'
)]
public function handle(Order $payload): void
{
    // handle
}
```

Read more about Error Channel in [related section](../../modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/).

## Final Failure Strategy

We can also define in case of unrecoverable failure, what should happen:

```php
#[RabbitConsumer(
    endpointId: 'orders_consumer',
    queueName: 'orders',
    finalFailureStrategy: FinalFailureStrategy::RESEND
)]
public function handle(Order $payload): void
{
    // handle
}
```

Read more about final failure strategy in [related section](../../modelling/recovering-tracing-and-monitoring/resiliency/final-failure-strategy.md).

## Deduplication

We can define custom deduplication key to ensure no same Message will be handled twice.

```php
#[Deduplicated('orderId')]
#[RabbitConsumer(
    endpointId: 'orders_consumer',
    queueName: 'orders'
)]
public function handle(Order $payload): void
{
    // handle
}
```

Read more about [deduplication in related section](../../modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md).
