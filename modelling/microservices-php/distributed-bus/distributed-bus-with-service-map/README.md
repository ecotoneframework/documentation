# Distributed Bus with Service Map

Cross-service messaging using a Service Map that supports multiple message channel providers (RabbitMQ, Amazon SQS, Redis, Kafka, and others) within a single application topology. Swap transports without changing application code.

**You'll know you need this when:**

* You're running 3+ microservices that need to exchange commands and events
* Different services use different message brokers (some RabbitMQ, some SQS) and you need them to communicate
* You're migrating from one broker to another and want to do it incrementally without rewriting application code
* Building custom inter-service messaging wiring for each service pair has become unsustainable

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
