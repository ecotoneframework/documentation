# Kafka Support

{% hint style="success" %}
This module is available as part of **Ecotone Enterprise.**
{% endhint %}

## Installation

```php
composer require ecotone/kafka
```

## Configuration

In order to use **Kafka Support** we need to add **KafkaBrokerConfiguration** to our **Dependency Container.**&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
# config/services.yaml
# You need to have RabbitMQ instance running on your localhost, or change DSN
    Ecotone\Kafka\Configuration\KafkaBrokerConfiguration:
        class: Ecotone\Kafka\Configuration\KafkaBrokerConfiguration
        arguments:
            bootstrapServers:
                - localhost:9094
```
{% endtab %}

{% tab title="Laravel" %}
```php
# Register AMQP Service in Provider

use Ecotone\Kafka\Configuration\KafkaBrokerConfiguration;

public function register()
{
     $this->app->singleton(KafkaBrokerConfiguration::class, function () {
         return new KafkaBrokerConfiguration(['localhost:9094']);
     });
}
```
{% endtab %}

{% tab title="Lite" %}
```php
use Ecotone\Kafka\Configuration\KafkaBrokerConfiguration;

$application = EcotoneLiteApplication::boostrap(
    [
        KafkaBrokerConfiguration::class => new KafkaBrokerConfiguration(['localhost:9094'])
    ]
);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
We register our **KafkaBrokerConfiguration** under the class name **Ecotone\Kafka\Configuration\KafkaBrokerConfiguration**. This will help Ecotone resolve it automatically, without any additional configuration.
{% endhint %}

## Message Channel

To create Kafka Backed [Message Channel](../modelling/asynchronous-handling/), we need to create [Service Context](../messaging/service-application-configuration.md).&#x20;

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

We can also customize the group id, which by default with following channel name:

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

## Custom Publisher and Consumer

To create [custom publisher or consumer](../modelling/microservices-php/) provide [Service Context](../messaging/service-application-configuration.md).

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

Then Publisher will be available for us in Dependency Container under **MessagePublisher** reference.

### Custom Consumer

To set up Consumer, consuming from given topics, all we need to do, is to mark given method with KafkaConsumer attribute:

```php
#[KafkaConsumer('ordersConsumers', 'orders')]
public function handle(string $payload, array $metadata): void
{
    // do something
}
```

Then we run it as any other [asynchronous consumer](../modelling/asynchronous-handling/), using **ordersConsumer** name.

### Custom Topic Configuration

We can also customize topic configuration. For example to create reference name for Consumers and publishers, which internally in Kafka will map to different name

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function distributedPublisher()
    {
        return TopicConfiguration::createWithReferenceName("orders", 'crm_orders');
    }
}
```
