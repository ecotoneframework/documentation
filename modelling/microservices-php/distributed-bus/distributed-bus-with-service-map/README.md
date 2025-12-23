# Distributed Bus with Service Map

Ecotone comes with [Message Channel abstraction](../../../asynchronous-handling/) which allows for easily switching from [different Providers](../../../asynchronous-handling/) like Amazon SQS, RabbitiMQ, Redis, Kafka and more.\
This abstraction is used for Service (application) level asynchronous communication like Asynchronous Message Handlers. However this abstraction can also be combined with Distributed Bus mechanism to enable cross Service Communication using **Service Map**.

{% hint style="success" %}
This functionality is available as part of **Ecotone Enterprise.**
{% endhint %}

## Distributed Service Map

It happens that communication between Services (Applications) is built using different Message Broker features to set up the topology. Which may require per feature configuration and provisioning, and in-depth knowledge about the Message Broker. This often end up as really complex solution, which becomes hard to understand and follow for Developers. When things becomes hard to understand, they become hard to change, as it raises the risk that potential modification may break something. As a result people try to avoid doing changes and development slows down.\
\
Therefore there is a need for different approach which keeps the things simple, easy to understand and change. Changes to the integration should not be scary, they should be straight forward and testable, so Developers can feel confidence in doing so. The best solution does not only make things simple to change, but also make things explicit, so just by looking people get more knowledge about the overal system design. And for this Ecotone comes with approach for Service to Service integration based on Service Ma&#x70;**.**

**Service Map** is a map of integrated Services (Applications), and points to specific Message Channels to which Messages for given Service should be sent:

<figure><img src="../../../../.gitbook/assets/service-map (1).png" alt=""><figcaption><p>Distributed Service Map</p></figcaption></figure>

In this approach **Message Channels (Pipes) are simple transport layer**, and the **routing is done on the Application (Endpoint)** level using **Service Map to make the decision**.

<figure><img src="../../../../.gitbook/assets/routing (1) (1).png" alt=""><figcaption><p>Distributed Bus making decision to which Message Channel send the Message</p></figcaption></figure>

Making Service available for integration is matter of adding it to the Service Map:

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              // Map commands to specific service
              ->withCommandMapping(
                        targetServiceName: "ticketService",
                        channelName: "distributed_ticket_service"
              )
              // Subscribe to events from other services
              ->withEventMapping(
                        channelName: "distributed_ticket_service",
                        subscriptionKeys: ["user.*", "order.created"],
              )
}
```

and defining implementation of the Message Channel:

```php
#[ServiceContext]
public function channels()
{
    return SqsBackedMessageChannelBuilder::create("distributed_ticket_service")
}
```

We may choose any Message Channel Provider we want, or even different providers depending on the integrated Service. This opens possibilities for using right tool for right job, as for example in one integration we could use Redis and for other one RabbitMQ or Kafka.

{% hint style="success" %}
Having the routing map on the Application level instead of Message Broker level means we avoid vendor-lock. In case of need to switch to different Message Broker Provider, we can simply change Message Channel implementation and our integration will continue to work.
{% endhint %}

Approach of treating Message Brokers as simple transport layer and doing the routing on the Application level to send to the right Message Channel follows [smart endpoint dump pipes](https://martinfowler.com/articles/microservices.html#SmartEndpointsAndDumbPipes) approach.\
\
This as a result make System easy to reason about and understand. Every developer can simply take look on the Map to understand where the Message will land. Adding new integration does not require specific Message Broker knowledge, as it all comes up to adding an Service to the Map and defining the Message Channel provider. Therefore Developers can add easily maintain and change such integration.

As we understand the concepts behind Distributed Bus with Service Map now, let's dive into practical example.

## How Command Distribution works

Let’s suppose **User Service** wants to create Ticket by sending Command to **Ticket Service.**\
\
In Ticket Service we will explicitly state that we allow given Command Handler to be executed in Distributed way. This makes it clear for everyone that we can't simply delete this Command Handler, as other Services may rely on this integration:

```php
#[Distributed]
#[CommandHandler("ticketService.createTicket")]
public function changeBillingDetails(CreateTicket $command) : void
{
    // create new Ticket
}
```

on the side of User Service we would call Distributed Bus for Command to do so:

```php
public function whenPersonRegistered($personId, DistributedBus $distributedBus)
{
    $distributedBus->convertAndSendCommand(
        targetServiceName: "ticketService",
        routingKey: "ticketService.createTicket",
        command: new CreateTicket($personId, "Call Customer to collect more details")
    );
}
```

**Our topology will look like this:**

<figure><img src="../../../../.gitbook/assets/createTicket.png" alt=""><figcaption><p>Topology of our Service to Service Communication</p></figcaption></figure>

As we can see above, we do have our two Services "**UserService"** and "**TicketService".**\
**Ticket Service** will be consuming incoming messages from "**distributed\_ticket\_service"** Messsage Channel. Therefore when we send Messages to ticketService we need to send it to that Message Channel, this will be done automatically by DistributedBus using Service Map.

In User Service let's then define Service Map using [ServiceContext](../../../../messaging/service-application-configuration.md) configuration:

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withCommandMapping(
                        targetServiceName: "ticketService",
                        channelName: "distributed_ticket_service"
              )
}
```

Now when we will send Command to ticketService it will land in **distributed\_ticket\_service** channel.

### Two level routing

Two level routing is a way to find target Service and within in Command Handler we want to execute:

```php
$distributedBus->convertAndSendCommand(
    targetServiceName: "ticketService",
    routingKey: "ticketService.createTicket",
    command: new CreateTicket($personId, "Call Customer to collect more details")
);
```

**"targetServiceName"** will be used to target specific Service therefore it will make use Message Channel defined in the Service Map.\
When Message will land in given Service, it will then use "**routingKey"** to target specific Command Handler within the Application.

<figure><img src="../../../../.gitbook/assets/real-topology (1).png" alt=""><figcaption><p>How Distributing Command works under the hood</p></figcaption></figure>

As you can see above, under the hood in target Service DistributedCommandHandler will be executed. This Handler is entrypoint to our Service, and will be responsible for triggering Command/Event Bus with given routing key.

## How Event Distribution works

Event distribution is a bit different from Command distribution. In case of Command we do have single Service that will receive the Message, in case of Events however there may be multiple of them.\
\
Let's expand our previous example to include Order Service, and our scenario is that whenever new User is registered in User Service, we will publish this event to both Ticket and Order Services.

<figure><img src="../../../../.gitbook/assets/default-publishing (1).png" alt=""><figcaption><p>Publishing Event to all known Services</p></figcaption></figure>

On the consumption part we will be marking our Event Handlers with Distributed:

```php
#[Distributed]
#[EventHandler("user.was_registered")]
public function when(UserRegistered $event) : void
{
    // do something
}
```

On the publishing side, we will be using publish event method with Distributed Bus:

```php
$distributedBus->convertAndPublishEvent(
    routingKey: "user.was_registered",
    event: new UserRegistered($userId)
);
```

Like you can see there is no **targetServiceName** in the parameters anymore (comparing to distributing Command), this is because Event may land in more than one Service. However we keep **routingKey** as this the name to which consuming Services subscribe (look EventHandler attribute parameter).

### Filtered Publishing

Filtered publishing allows for optimalization in publishing. This way we can publish Events only to the Services that are actually interested in those.

<figure><img src="../../../../.gitbook/assets/publishers (1).png" alt=""><figcaption><p>Publishing based on Service Map subscribition keys</p></figcaption></figure>

To configure map with subscription keys, we will be using **withEventMapping()** method in Service Map configuration:

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              // Event subscriptions with filtering
              ->withEventMapping(
                        channelName: "distributed_ticket_service",
                        subscriptionKeys: ["userService.account.*"],
              )
              ->withEventMapping(
                        channelName: "distributed_order_service",
                        subscriptionKeys: ["userService.address.changed"],
              )
}
```

Subscription keys is array, therefore we may put multiple subscription routing keys if needed.\
Subscription keys can point to exact event name: "**userService.address.changed"**\
Or they may use wild card: "**userService.account.\*"**

## Sharing Events using Streaming Channels

Streaming channels provide a powerful way to share events between services. Unlike traditional message queues where each message is consumed by a single consumer, streaming channels allow multiple services to independently consume the same stream of events. Each service tracks its own position in the stream, enabling features like event replay and independent consumption rates.

{% hint style="success" %}
Streaming channels are ideal for event-driven architectures where multiple services need to react to the same events independently.\
As Event can be published once, but consumed multiple times.
{% endhint %}

### Benefits of Streaming Channels for Event Distribution

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

## **Shared Service Map**

When Service Map is defined as separate shared library. It becomes explicit what Events is given Service interested in. This also makes the process of subscribing to new Event visible for everyone, therefore we avoid hidden coupling that could lead to broken integration.

Depending on how we share the events Queue vs Streaming, we may want to do additional changes to the Service Map

{% hint style="success" %}
When Service Map is defined as separate shared library. It becomes explicit what Events is given Service interested in. This also makes the process of subscribing to new Event visible for everyone, therefore we avoid hidden coupling that could lead to broken integration.
{% endhint %}

### Explicit Mapping

To be as explicit as it's possible, we can use direct mapping for subscription keys

```php
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "order_service_inbound_events",
                        subscriptionKeys: ["ticket_service.management.*"],
              )
}
```

This ensures that we won't receive Events that are not meant to be directed to given Service.

### Non Explicit Mapping

In some cases, especially with existing Services, explicit mapping may not be possible, in this situations, we may need to use additional functionality to add filtering.

#### Queue Channel

In Queue Channel approach, it's external Service that owns that Channel, Publisher is only delivering his own Events there as part of Service Map contract. However, if we would put about "\*" for receiving events, then if Publishing Service would also be the owner of the Channel we would republish his own Messages back to that Channel.

To avoid that we could use **excludePublishingServices:**

```php
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "order_service_inbound_events",
                        subscriptionKeys: ["*"],
                        excludePublishingServices: ["ticket_service"]
              )
}
```

#### Streaming Channel

With Streaming Channel, the publishing Service, will actually be the owner of the Channel to which we want to deliver the Message. Therefore for this, if we are about to use the "\*" to publish all the Messages there, reusing the Service Map in other Service would deliver other Events there too. <br>

To avoid that and to be explicit that we want there only Events from particular Service, we can use **includePublishingServices:**

<pre class="language-php"><code class="lang-php"><strong>public function serviceMap(): DistributedServiceMap
</strong>{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "shared_order_events",
                        subscriptionKeys: ["*"],
                        includePublishingServices: ["order_service"]
              )
}
</code></pre>
