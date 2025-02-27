# Usage

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

## Custom Publisher

To create [custom publisher or consumer](../../modelling/microservices-php/) provide [Service Context](../../messaging/service-application-configuration.md).

{% hint style="success" %}
Custom Publishers and Consumers are great for building integrations for existing infrastructure or setting up a customized way to communicate between applications. With this you can take over the control of what is published and how it's consumed.
{% endhint %}

### Custom Publisher

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function distributedPublisher()
    {
        return KafkaPublisherConfiguration::createWithDefaults(
            topicName: 'orders'
        );
    }
}
```

Then Publisher will be available for us in Dependency Container under **MessagePublisher** reference.\
This will make it available in your dependency container under **MessagePublisher** name.

### Providing custom rdkafka configuration

You can modify your Message Publisher with specific [rdkafka configuration](https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md):

```php
#[ServiceContext] 
public function distributedPublisher()
{
    return KafkaPublisherConfiguration::createWithDefaults(
        topicName: 'orders'
    )
        ->setConfiguration('request.required.acks', '1');
}
```

## Custom Consumer

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
