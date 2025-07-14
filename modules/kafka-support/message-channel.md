# Message Channel

To understand the use case behind Message Channels read [Asynchronous Processing](../../modelling/asynchronous-handling/asynchronous-message-handlers.md) section for Application level processing and [Distributed Bus](../../modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) section for cross application communication.

## Message Channel

To create Kafka Backed [Message Channel](../../modelling/asynchronous-handling/), we need to create [Service Context](../../messaging/service-application-configuration.md).&#x20;

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function orderChannel()
    {
        return KafkaMessageChannelBuilder::create("orders");
    }
}
```

Now `orders` channel will be available in our Messaging System.&#x20;

{% hint style="success" %}
Message Channels simplify to the maximum integration with Message Broker. \
From application perspective all we need to do, is to provide channel implementation.\
Ecotone will take care of whole publishing and consuming part.&#x20;
{% endhint %}

### Customize Topic Name

By default the queue name will follow channel name, which in above example will be "orders".\
However we can use "orders" as reference name in our Application, yet name queue differently:

```php
#[ServiceContext] 
public function orderChannel()
{
    return KafkaMessageChannelBuilder::create(
        channelName: "orders",
        topicName: "crm_orders"
    );
}
```

### Customize Group Id

We can also customize the group id, which by default following channel name:

```php
#[ServiceContext] 
public function orderChannel()
{
    return KafkaMessageChannelBuilder::create(
        channelName: "orders",
        groupId: "crm_application"
    );
}
```

{% hint style="warning" %}
Position of Message Consumer is tracked against given group id.. Depending on retention policy, changing group id for existing Message Channel may result in re-delivering messages.
{% endhint %}

## Final Failure Strategy

To define [Final Failure Strategy](../../modelling/recovering-tracing-and-monitoring/resiliency/final-failure-strategy.md):

```php
#[ServiceContext] 
public function orderChannel()
{
    return KafkaMessageChannelBuilder::create(
        channelName: "orders",
        groupId: "crm_application"
    )
        ->withFinalFailureStrategy(FinalFailureStrategy::RESEND);
}
```
