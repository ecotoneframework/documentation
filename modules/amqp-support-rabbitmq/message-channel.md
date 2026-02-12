# Message Channel

To understand the use case behind Message Channels read [Asynchronous Processing](../../modelling/asynchronous-handling/asynchronous-message-handlers.md) section for Application level processing and [Distributed Bus](../../modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) section for cross application communication.

## Message Channel

To create AMQP Backed [Message Channel](../../modelling/asynchronous-handling/) (RabbitMQ Channel), we need to create [Service Context](../../messaging/service-application-configuration.md).

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function orderChannel()
    {
        return AmqpBackedMessageChannelBuilder::create("orders");
    }
}
```

Now `orders` channel will be available in our Messaging System.

{% hint style="success" %}
Message Channels simplify to the maximum integration with Message Broker.\
From application perspective all we need to do, is to provide channel implementation.\
Ecotone will take care of whole publishing and consuming part.
{% endhint %}

### Message Channel Configuration

```php
AmqpBackedMessageChannelBuilder::create("orders")
    ->withAutoDeclare(false) // do not auto declare queue
    ->withDefaultTimeToLive(1000) // limit TTL of messages
    ->withDefaultDeliveryDelay(1000) // delay messages by default
    ->withFinalFailureStrategy(FinalFailureStrategy:RESEND) // final failure strategy
```

### Customize Queue Name

By default the queue name will follow channel name, which in above example will be "orders".\
However we can use "orders" as reference name in our Application, yet name queue differently:

```php
#[ServiceContext] 
public function orderChannel()
{
    return AmqpBackedMessageChannelBuilder::create(
        channelName: "orders",
        queueName: "crm_orders"
    );
}
```

## Usage

Then Message Channels can be used as follows to make [Message Handler asynchronous](../../modelling/asynchronous-handling/asynchronous-message-handlers.md):

```php
#[Asynchronous("orders")]
#[EventHandler(endpointId: "order_was_placed")
public function when(OrderWasPlaced $event) : void
{
   // do something with $event
}
```

## RabbitMQ Streaming Channel

Persistent event streaming on existing RabbitMQ infrastructure -- Kafka-like semantics without adding Kafka. Multiple independent consumers read from the same stream, each tracking their own position. Events are persisted and can be replayed from any offset.

**You'll know you need this when:**

* You need multiple independent consumers reading the same event stream, each tracking their own position
* You have existing RabbitMQ infrastructure and don't want the operational overhead of adding Kafka
* You need event replay capabilities where consumers can re-read from specific positions
* Standard RabbitMQ queues (where consumed messages disappear) don't fit your event-driven architecture

{% hint style="success" %}
This functionality is available as part of **Ecotone Enterprise.**
{% endhint %}

### Requirements

To use RabbitMQ Streaming Channels, you need to install:

1. **Ecotone DBAL Module** - Required for storing consumer position tracking

```bash
composer require ecotone/dbal
```

2. **AmqpLib Connection Factory** - Required for RabbitMQ streaming support (see [Connection Factory Setup](message-channel.md#connection-factory-setup) below)

{% hint style="info" %}
The DBAL module is used to persist the consumer position (offset) in the database. This ensures that if your application restarts, consumers can resume from where they left off.
{% endhint %}

### Key Features

* **Multiple Independent Consumers**: Each consumer maintains its own position in the stream
* **Event Replay**: Consumers can start from any position (first, last, next, or specific offset)
* **Durability**: Events are persisted and can be consumed multiple times
* **Position Tracking**: Automatic tracking of consumer position with configurable commit intervals

### Basic Configuration

To create a RabbitMQ Streaming Channel, you need to:

1. Create a stream queue using `AmqpQueue::createStreamQueue()`
2. Configure the streaming channel with `AmqpStreamChannelBuilder`

```php
use Ecotone\Amqp\AmqpQueue;
use Ecotone\Amqp\AmqpStreamChannelBuilder;
use Enqueue\AmqpLib\AmqpConnectionFactory;

class MessagingConfiguration
{
    #[ServiceContext]
    public function eventStreamChannel()
    {
        return [
            // 1. Create the stream queue
            AmqpQueue::createStreamQueue("events_stream"),

            // 2. Configure the streaming channel
            AmqpStreamChannelBuilder::create(
                channelName: "events",
                startPosition: "first", // Where to start consuming
                amqpConnectionReferenceName: AmqpConnectionFactory::class,
                queueName: "events_stream",
                messageGroupId: "my-service-consumer" // Unique consumer group ID
            )
        ];
    }
}
```

{% hint style="warning" %}
**Important**: RabbitMQ Streaming Channels require the **AmqpLib** connection factory (`Enqueue\AmqpLib\AmqpConnectionFactory`). The AmqpExt connection factory does not support streaming features.
{% endhint %}

### Connection Factory Setup

Make sure you're using the AmqpLib connection factory in your dependency container:

{% tabs %}
{% tab title="Symfony" %}
```php
# config/services.yaml
    Enqueue\AmqpLib\AmqpConnectionFactory:
        class: Enqueue\AmqpLib\AmqpConnectionFactory
        arguments:
            - "amqp://guest:guest@localhost:5672//"
```
{% endtab %}

{% tab title="Laravel" %}
```php
# Register AMQP Service in Provider

use Enqueue\AmqpLib\AmqpConnectionFactory;

public function register()
{
    $this->app->singleton(AmqpConnectionFactory::class, function () {
        return new AmqpConnectionFactory("amqp://guest:guest@localhost:5672//");
    });
}
```
{% endtab %}

{% tab title="Lite" %}
```php
use Enqueue\AmqpLib\AmqpConnectionFactory;

$application = EcotoneLite::bootstrap(
    [AmqpConnectionFactory::class => new AmqpConnectionFactory("amqp://guest:guest@localhost:5672//")]
);
```
{% endtab %}
{% endtabs %}

### Start Position Options

The `startPosition` parameter controls where the consumer begins reading from the stream:

* **`"first"`** - Start from the beginning of the stream (replay all events)
* **`"last"`** - Start from the end of the stream (skip existing events)
* **`"next"`** - Start from the next new message (default behavior)
* **Specific offset** - Start from a specific offset number (e.g., `"12345"`)

```php
// Start from the beginning (replay all events)
AmqpStreamChannelBuilder::create(
    channelName: "events",
    startPosition: "first",
    queueName: "events_stream",
    messageGroupId: "replay-consumer"
)

// Start from the end (only new events)
AmqpStreamChannelBuilder::create(
    channelName: "events",
    startPosition: "last",
    queueName: "events_stream",
    messageGroupId: "new-events-consumer"
)
```

### Message Group ID (Consumer Groups)

The `messageGroupId` is a unique identifier for each consumer group. Multiple consumers with the same `messageGroupId` will share the same position in the stream, while consumers with different IDs track their positions independently.

```php
// Service 1 - Ticket Service
AmqpStreamChannelBuilder::create(
    channelName: "shared_events",
    queueName: "events_stream",
    messageGroupId: "ticket-service-consumer" // Unique ID
)

// Service 2 - Order Service
AmqpStreamChannelBuilder::create(
    channelName: "shared_events",
    queueName: "events_stream",
    messageGroupId: "order-service-consumer" // Different ID
)
```

Both services consume from the same stream (`events_stream`) but track their positions independently.

### Advanced Configuration

#### Commit Interval

Controls how often the consumer position is committed. Lower values are safer but slower, higher values improve performance but may cause reprocessing on failure.

```php
AmqpStreamChannelBuilder::create(
    channelName: "events",
    queueName: "events_stream",
    messageGroupId: "my-consumer"
)
    ->withCommitInterval(100) // Commit every 100 messages (default: 100)
```

**How it works:**

* `commitInterval=1`: Commit after every message (safest, slowest)
* `commitInterval=100`: Commit after every 100 messages (better performance)
* The last message in a batch is always committed, even if the interval hasn't been reached

#### Prefetch Count

Controls how many unacknowledged messages RabbitMQ will deliver to the consumer.

```php
AmqpStreamChannelBuilder::create(
    channelName: "events",
    queueName: "events_stream",
    messageGroupId: "my-consumer"
)
    ->withPrefetchCount(50) // Prefetch 50 messages (default: 100)
```

**Guidelines:**

* Lower values (e.g., 1-10): Better flow control, lower throughput
* Higher values (e.g., 50-100): Higher throughput, more memory usage

### Complete Example

Here's a complete example showing how to set up a streaming channel for distributed event sharing:

```php
use Ecotone\Amqp\AmqpQueue;
use Ecotone\Amqp\AmqpStreamChannelBuilder;
use Enqueue\AmqpLib\AmqpConnectionFactory;

class MessagingConfiguration
{
    #[ServiceContext]
    public function distributedEventStream()
    {
        return [
            // Create the stream queue
            AmqpQueue::createStreamQueue("distributed_events_stream"),

            // Configure streaming channel with optimized settings
            AmqpStreamChannelBuilder::create(
                channelName: "distributed_events",
                startPosition: "first",
                queueName: "distributed_events_stream",
                messageGroupId: "ticket-service-consumer"
            )
                ->withCommitInterval(100)  // Commit every 100 messages
                ->withPrefetchCount(50)    // Prefetch 50 messages
        ];
    }
}
```

### Usage with Distributed Bus

Streaming channels work seamlessly with the [Distributed Bus](../../modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/):

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "distributed_user_events",
                        subscriptionKeys: ["user.*"],
              )
}

#[Distributed]
#[EventHandler("user.registered")]
public function onUserRegistered(UserRegistered $event): void
{
    // Handle event - can replay from beginning if needed
}
```

For more examples of using streaming channels with Distributed Bus, see the Distributed Bus with Service Map documentation.
