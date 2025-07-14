# Message Channel

To understand the use case behind Message Channels read [Asynchronous Processing](../../modelling/asynchronous-handling/asynchronous-message-handlers.md) section for Application level processing and [Distributed Bus](../../modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) section for cross application communication.

## Message Channel

To create AMQP Backed [Message Channel](../../modelling/asynchronous-handling/) (RabbitMQ Channel), we need to create [Service Context](../../messaging/service-application-configuration.md).&#x20;

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

Now `orders` channel will be available in our Messaging System.&#x20;

{% hint style="success" %}
Message Channels simplify to the maximum integration with Message Broker. \
From application perspective all we need to do, is to provide channel implementation.\
Ecotone will take care of whole publishing and consuming part.&#x20;
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
