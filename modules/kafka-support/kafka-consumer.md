---
description: Kafka consumer for event streaming in Ecotone
---

# Kafka Consumer

To quickly get up and running with consuming existing Topics, we can use Kafka Consumer feature.\
Simply by marking given method with Kafka Consumer attribute, we are getting access to asynchronous process that will now run and consume Message from defined Topic.

## Kafka Consumer

To set up Consumer, consuming from given topics, all we need to do, is to mark given method with KafkaConsumer attribute:

```php
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders']
)]
public function handle(string $payload, array $metadata): void
{
    // do something
}
```

Then we run it as any other [asynchronous consumer](../../modelling/asynchronous-handling/), using **orderConsumer** name.

### Providing group id

By default Consumer Group id will be same as endpoint id, however we can provide customized name if needed:

```php
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders'],
    groupId: 'order_subscriber'
)]
public function handle(string $payload, array $metadata): void
{
    // do something
}
```

### Providing custom rdkafka configuration

You can modify your Message Consumer with specific [rdkafka configuration](https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md):

```php
#[ServiceContext] 
public function distributedPublisher()
{
    return KafkaConsumerConfiguration::createWithDefaults(
        endpointId: 'orderConsumers'
    )
        ->setConfiguration('auto.offset.reset', 'earliest');
}
```

## Kafka Headers

We can accesss specific Kafka Headers using standard [Ecotone's metadata](../../modelling/event-sourcing/event-sourcing-introduction/working-with-metadata.md) mechanism

* **kafka\_topic** - Topic name for incoming message
* **kafka\_partition** - Partition of incoming message
* **kafka\_offset** - Offset of Message Consumer

```php
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders']
)]
public function handle(
    string $payload, 
    #[Header("kafka_topic")] string $topicName,
): void
{
    // do something
}
```

## Automatic Conversion

If we have our Custom Conversion or [JMS Module ](../jms-converter.md)installed, then we can leverage automatic conversion:

```php
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders']
)]
public function handle(Order $payload): void
{
    // do something
}
```

Read more about Conversion in [related section](../../messaging/conversion/conversion/).

### Default Content Type

Messages published by external systems may come without contentType header, and then Ecotone does not know from which format the payload should be deserialized. If our Topic follows known format, we can state it explicitly using ContentType attribute:

```php
#[ContentType('application/json')]
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders']
)]
public function handle(Order $payload): void
{
    // do something
}
```

If Message already carries contentType header, it stays untouched — the attribute works as a default. To always enforce given Media Type, we can use **replaceIfExists** flag:

```php
#[ContentType('application/json', replaceIfExists: true)]
```

Read more about Default Content Type in [related section](../../messaging/conversion/conversion/payload-conversion.md#default-content-type-for-consumed-messages).

## Instant Retry&#x20;

In case of failure we may try to recover instantly. To do so we can provide **InstantRetry** attribute:

```php
#[InstantRetry(retryTimes: 3)]
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders']
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
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders']
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
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders'],
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
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders']
)]
public function handle(Order $payload): void
{
    // handle
}
```

Read more about [deduplication in related section](../../modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md).
