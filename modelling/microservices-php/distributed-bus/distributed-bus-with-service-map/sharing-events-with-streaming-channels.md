# Sharing Events with Streaming Channels

## Sharing Events using Streaming Channels

Streaming channels provide a powerful way to share events between services. Unlike traditional message queues where each message is consumed by a single consumer, streaming channels allow multiple services to independently consume the same stream of events. Each service tracks its own position in the stream, enabling features like event replay and independent consumption rates.

{% hint style="success" %}
Streaming channels are ideal for event-driven architectures where multiple services need to react to the same events independently.\
As Event can be published once, but consumed multiple times.
{% endhint %}

## Architecture

To understand the difference better, let's assume we are using Queue Channel first, and we do have Service Map as follows:

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
        ->withEventMapping(
            channelName: "ticket_events",
            subscriptionKeys: ["user.*"],
        )
        ->withEventMapping(
            channelName: "order_events",
            subscriptionKeys: ["user.*"],
        );
}
```

This will map to this flow:

<figure><img src="../../../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

Like we can see, this requires delivery of same Events to both Channels ticket and order events. \
With Streaming Channels however, we can decide that publisher "announces" given set of Events, and let everyone joined the subscription instead:

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
        ->withEventMapping(
            channelName: "user_events",
            subscriptionKeys: ["user.*"],
        );
}
```

This will map to this flow:

<figure><img src="../../../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Then everyone can join the consumption from user\_events, as Messages in Streaming Channels are not removed after consumption. Therefore we do have one shared event log, to be consumed by different Services.

## Benefits of Streaming Channels for Event Distribution

* **Independent Consumption**: Each service maintains its own position in the event stream
* **Event Replay**: Services can replay events from any point in the stream
* **Scalability**: Multiple consumers can read from the same stream without affecting each other
* **Durability**: Events are persisted and can be consumed multiple times
* **Decoupling**: Publishers don't need to know about consumers

### In Memory Streaming Channel

For testing and development, you can use in-memory streaming channels:

```php
#[ServiceContext]
public function distributedEventChannel()
{
    return SimpleMessageChannelBuilder::createStreamingChannel(
        messageChannelName: "distributed_events",
        messageGroupId: "my-service-consumer-group"
    );
}

#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "distributed_events",
                        subscriptionKeys: ["user.*", "order.*"],
              )
}
```

Each service consuming from the same streaming channel should use a unique `messageGroupId` to track its position independently.

### RabbitMQ Streaming Channel

RabbitMQ Streams provide high-throughput, persistent event streaming:

```php
#[ServiceContext]
public function distributedEventChannel()
{
    return [
        // Configure streaming channel
        AmqpStreamChannelBuilder::create(
            channelName: "distributed_events",
            messageGroupId: "ticket-service-consumer" // Unique consumer group
        )
            ->withCommitInterval(100) // Commit position every 100 messages
    ];
}

#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "distributed_events",
                        subscriptionKeys: ["*"],
              )
}
```

**Configuration Options:**

* **messageGroupId**: Unique identifier for this consumer group - each service should have its own
* **commitInterval**: How often to commit the consumer position (in number of messages)

### Kafka Streaming Channel

Kafka provides distributed, fault-tolerant event streaming:

```php
#[ServiceContext]
public function distributedEventChannel()
{
    return KafkaMessageChannelBuilder::create(
        channelName: "distributed_events",
        topicName: "distributed_events_topic",
        messageGroupId: "order-service-consumer" // Kafka consumer group
    )
        ->withCommitInterval(50); // Commit offset every 50 messages
}

#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "distributed_events",
                        subscriptionKeys: ["user.*", "ticket.*"],
              )
}
```

**Configuration Options:**

* **topicName**: The Kafka topic to publish/consume from
* **messageGroupId**: Kafka consumer group ID - each service should have its own
* **commitInterval**: How often to commit the offset (in number of messages)

### Multi-Service Example

Here's a complete example showing how multiple services can share events using a streaming channel:

**Shared Service Map and Channel**

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "shared_user_events",
                        subscriptionKeys: ["user.*"],
              )
}
```

**Publisher Service (User Service):**

```php
// Publishing events
public function registerUser(DistributedBus $distributedBus)
{
    $distributedBus->convertAndPublishEvent(
        routingKey: "user.registered",
        event: new UserRegistered($userId, $email)
    );
}
```

```php
#[ServiceContext]
public function eventStreamChannel()
{
    return AmqpStreamChannelBuilder::create(
        channelName: "shared_user_events",
        messageGroupId: "user-service-consumer" // Unique consumer group per Service
    );
}
```

**Consumer Service 1 (Ticket Service):**

```php
#[Distributed]
#[EventHandler("user.registered")]
public function onUserRegistered(UserRegistered $event): void
{
    // Create welcome ticket
}
```

```php
#[ServiceContext]
public function eventStreamChannel()
{
    return AmqpStreamChannelBuilder::create(
        channelName: "shared_user_events",
        messageGroupId: "ticket-service-consumer" // Unique consumer group per Service
    )
        ->withCommitInterval(100);
}
```

**Consumer Service 2 (Order Service):**

```php
#[Distributed]
#[EventHandler("user.registered")]
public function onUserRegistered(UserRegistered $event): void
{
    // Initialize user's order history
}
```

```php
#[ServiceContext]
public function eventStreamChannel()
{
    return AmqpStreamChannelBuilder::create(
        channelName: "shared_user_events",
        messageGroupId: "order-service-consumer" // Unique consumer group per Service
    )
        ->withCommitInterval(100);
}
```

{% hint style="info" %}
Each consuming service uses the same `queueName` (RabbitMQ) or `topicName` (Kafka) but with a different `messageGroupId`. This allows them to consume the same events independently, each tracking their own position in the stream.
{% endhint %}
