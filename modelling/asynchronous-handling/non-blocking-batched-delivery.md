---
description: Batched, promise-based message publishing that multiplies throughput while keeping delivery guarantees
---

# Non-blocking Batched Delivery

When your application publishes messages to a Message Broker, each send is a full network round trip: serialize, write, and block until the Broker confirms. Publish a thousand messages and you pay for a thousand round trips, one after another.

Non-blocking Batched Delivery changes that equation. Messages are fired to the Broker without waiting for individual confirmations, multiple messages are combined into single Broker writes, and confirmations are collected once for the whole set - right before your transaction commits. The result is a multiplier on publishing throughput, without giving up a single delivery guarantee.

**You'll know you need this when:**

* A single Command results in many Events, and publishing them one by one dominates the request time
* You run imports, migrations or ETL jobs that push thousands of messages to a Broker
* Your Outbox or high-volume workflow is bottlenecked on publishing latency, not on processing
* You want fire-and-forget publishing speed, yet nothing may be silently lost

{% hint style="success" %}
Non-blocking Batched Delivery is available as part of **Ecotone Enterprise.**
{% endhint %}

## How it works

With synchronous publishing, every message follows the pattern: send, wait for Broker confirmation, send the next one. The waiting dominates - the Broker is mostly idle while your application blocks on network latency.

With Asynchronous Publishing enabled, Ecotone:

1. **Batches** - messages published within the same execution scope are combined and written to the Broker together (a single multi-row insert for DBAL, a batched publish for RabbitMQ, one batch request per 10 messages for SQS, a single script execution for Redis, pipelined delivery for Kafka)
2. **Defers confirmation** - instead of blocking per message, delivery confirmations are collected in the background and awaited once
3. **Awaits before commit** - all outstanding confirmations are resolved before your transaction commits, so a successful Command execution means every published message is confirmed by the Broker

This applies to all Broker providers: **RabbitMQ (AMQP), Kafka, Amazon SQS, Redis and Database Channels (DBAL)**.

## Enabling Asynchronous Publishing

Asynchronous Publishing is enabled on the Message Channel or Message Publisher level:

```php
#[ServiceContext]
public function orderChannel()
{
    return AmqpBackedMessageChannelBuilder::create("orders")
                ->withAsyncPublishing();
}
```

and for Message Publisher:

```php
#[ServiceContext]
public function messagePublisher()
{
    return AmqpMessagePublisherConfiguration::create()
                ->withDefaultRoutingKey("orders")
                ->withAsyncPublishing();
}
```

The same `withAsyncPublishing()` method is available on `KafkaMessageChannelBuilder`, `SqsBackedMessageChannelBuilder`, `RedisBackedMessageChannelBuilder`, `DbalBackedMessageChannelBuilder` and their corresponding Message Publisher configurations.

## Publishing from Business Code

The most common scenario requires no API changes at all. When your Command Handler publishes Events to an asynchronously published channel, Ecotone collects them and delivers them as one batch:

```php
#[CommandHandler]
public function placeOrder(PlaceOrder $command, EventBus $eventBus): void
{
    // each Event goes to the "orders" channel
    $eventBus->publish(new OrderWasPlaced($command->orderId));
    $eventBus->publish(new PaymentWasRequested($command->orderId));
    $eventBus->publish(new NotificationWasScheduled($command->orderId));
}
```

All three Events are written to the Broker in a single batched operation, and their confirmations are awaited together before the Command Bus returns. Your business code stays exactly the same - enabling `withAsyncPublishing()` on the channel is the only change.

## Publishing with Futures

For explicit control, `MessagePublisher` exposes `asyncPublish`, which fires the message and returns a `Future`:

```php
$future = $messagePublisher->asyncPublish($orderData); // 1

// do other work while the Broker processes the delivery

$future->resolve(); // 2
```

1. The message is sent to the Broker immediately, but the confirmation is not awaited
2. `resolve()` awaits the delivery confirmation - it throws `PublishingFailedException` if the Broker rejected the message

This enables pipelining: fire many publishes, let the Broker work on all of them concurrently, then resolve the Futures at the end:

```php
$futures = [];
foreach ($chunkedOrders as $batch) {
    $futures[] = $messagePublisher->asyncPublish($batch);
}

foreach ($futures as $future) {
    $future->resolve();
}
```

{% hint style="info" %}
You never risk losing a message by forgetting to resolve a Future. Ecotone awaits all unresolved deliveries before the surrounding transaction commits, and flushes any remaining ones on application shutdown.
{% endhint %}

## Batch Messages

To publish a set of messages as one explicit unit, use `BatchMessage`:

```php
$messagePublisher->asyncPublish(
    BatchMessage::constructEmpty()
        ->append($firstOrder)
        ->append($secondOrder, ['priority' => '5']) // 1
        ->append($reminder, [MessageHeaders::DELIVERY_DELAY => 60000]) // 2
        ->append($liveUpdate, [MessageHeaders::TIME_TO_LIVE => 5000]) // 3
)->resolve();
```

1. Each entry carries its own headers
2. Entries can be individually delayed
3. Entries can individually expire

The whole batch is delivered to the Broker in a single operation, yet each entry keeps its own metadata, delay and time to live.

## Delivery Guarantees

Speed without safety would be no gain at all. Asynchronous Publishing keeps the full set of Ecotone's delivery guarantees:

* **Confirmed before commit** - all pending deliveries are awaited before the transaction commits. A Command that finished successfully means every published message is safely stored in the Broker
* **Per-message failure attribution** - when part of a batch fails, Ecotone knows exactly which messages failed. Retries redeliver only the failed ones - already delivered messages are never duplicated
* **Error Channel routing** - messages that exhaust retries are routed to your [Error Channel or Dead Letter](../recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/README.md) individually, each as a separate, replayable message
* **No silent loss** - deliveries that were never explicitly resolved are awaited at the transaction boundary and flushed on shutdown, with failures logged and routed

For the details of send-path resiliency, see [Resilient Sending](../recovering-tracing-and-monitoring/resiliency/resilient-sending.md).

## The Throughput Multiplier

Publishing 1000 messages to a Broker, single synchronous sends vs batched Asynchronous Publishing, measured with Ecotone's benchmark suite on a local Docker setup:

| Provider | 1000 synchronous sends | Batched Asynchronous Publishing | Multiplier |
| --- | --- | --- | --- |
| Kafka | 292 ms | 22 ms | **~13x** |
| Amazon SQS | 1776 ms | 165 ms | **~10x** |
| Database (DBAL) | 435 ms | 48 ms | **~9x** |
| RabbitMQ (AMQP) | 240 ms | 35 ms | **~7x** |
| Redis | 77 ms | 24 ms | **~3x** |

These numbers come from a local network setup, where round trips are cheapest. In production, where the Broker sits behind real network latency, every eliminated round trip is worth more - the multiplier grows with the distance to your Broker and with the volume published per scope.

## Materials

### Links

* [Message Publisher](../microservices-php/message-publisher.md) \[Documentation]
* [Delivery Semantics and Guarantees](../recovering-tracing-and-monitoring/delivery-semantics-and-guarantees.md) \[Documentation]
