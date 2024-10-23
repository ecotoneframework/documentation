---
description: Asynchronous PHP RabbitMQ
---

# RabbitMQ Support

## Installation

```php
composer require ecotone/amqp
```

### Module Powered By

[Enqueue](https://github.com/php-enqueue/enqueue-dev) solid and powerful abstraction over asynchronous queues.

## Configuration

In order to use `AMQP Support` we need to add `ConnectionFactory` to our `Dependency Container.`&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
# config/services.yaml
# You need to have RabbitMQ instance running on your localhost, or change DSN
    Enqueue\AmqpExt\AmqpConnectionFactory:
        class: Enqueue\AmqpExt\AmqpConnectionFactory
        arguments:
            - "amqp://guest:guest@localhost:5672//"
```
{% endtab %}

{% tab title="Laravel" %}
```php
# Register AMQP Service in Provider

use Enqueue\AmqpExt\AmqpConnectionFactory;

public function register()
{
     $this->app->singleton(AmqpConnectionFactory::class, function () {
         return new AmqpConnectionFactory("amqp+lib://guest:guest@localhost:5672//");
     });
}
```
{% endtab %}

{% tab title="Lite" %}
```php
use Enqueue\AmqpExt\AmqpConnectionFactory;

$application = EcotoneLiteApplication::boostrap(
    [
        AmqpConnectionFactory::class => new AmqpConnectionFactory("amqp+lib://guest:guest@localhost:5672//")
    ]
);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
We register our AmqpConnection under the class name `Enqueue\AmqpExt\AmqpConnectionFactory.` This will help Ecotone resolve it automatically, without any additional configuration.
{% endhint %}

## Message Channel

To create AMQP Backed [Message Channel](../modelling/asynchronous-handling/) (RabbitMQ Channel), we need to create [Service Context](../messaging/service-application-configuration.md).&#x20;

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

### Message Channel Configuration

```php
AmqpBackedMessageChannelBuilder::create("orders")
    ->withAutoDeclare(false) // do not auto declare queue
    ->withDefaultTimeToLive(1000) // limit TTL of messages
    ->withDefaultDeliveryDelay(1000) // delay messages by default
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

## Distributed Publisher and Consumer

To create [distributed publisher or consumer](../modelling/microservices-php/) provide [Service Context](../messaging/service-application-configuration.md).

### Distributed Publisher

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function distributedPublisher()
    {
        return AmqpDistributedBusConfiguration::createPublisher();
    }
}
```

### Distributed Consumer

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function distributedConsumer()
    {
        return AmqpDistributedBusConfiguration::createConsumer();
    }
}
```

## Message Publisher

If you want to publish Message directly to Exchange, you may use of `Publisher.`

```php
class AMQPConfiguration
{
    #[ServiceContext] 
    public function registerAmqpConfig()
    {
        return 
            AmqpMessagePublisherConfiguration::create(
                MessagePublisher::class, // 1
                "delivery", // 2
                "application/json" // 3
            );
    }
}
```

1. `Reference name` -  Name under which it will be available in Dependency Container.
2. `Exchange name` - Name of exchange where Message should be published
3. `Default Conversion [Optional]` - Default type, payload will be converted to.

### Publisher Configuration

```php
AmqpMessagePublisherConfiguration::create(
    MessagePublisher::class,
    "delivery"
)
    ->withDefaultPersistentDelivery(true) // 1
    ->withDefaultRoutingKey("someKey") // 2
    ->withRoutingKeyFromHeader("routingKey") // 3
    ->withHeaderMapper("application.*") // 4
```

1. `withDefaultPersistentDelivery` - should AMQP messages be `persistent`_._
2. `withDefaultRoutingKey` - default routing key added to AMQP message&#x20;
3. `withRoutingKeyFromHeader` - should routing key be retrieved from header with name
4. `withHeaderMapper` - On default headers are not send with AMQP message. You map provide mapping for headers that should be mapped to `AMQP Message`

## Message Consumer

We can bind given method as Message Consumer

```php
#[MessageConsumer('orders_consumer')] // name of endpoint id
public function handle(string $payload): void
{
    // do something
}
```

To connect consumer directly to a AMQP Queue, we need to provide `Ecotone` with information, how the Queue is configured.&#x20;

```php
class AmqpConfiguration
{
    #[ServiceContext] 
    public function registerAmqpConfig(): array
    {
        return [
            AmqpQueue::createWith("orders"), // 1
            AmqpExchange::createDirectExchange("system"), // 2
            AmqpBinding::createFromNames("system", "orders", "placeOrder"), // 3
            AmqpMessageConsumerConfiguration::create("orders_consumer", "orders") // 4
        ];
    }
}
```

1. `AmqpQueue::createWith(string $name)` - Registers Queue with specific name
2. `AmqpExchange::create*(string $name)` - Registers of given type with specific name
3. `AmqpBinding::createFromName(string $exchangeName, string $queueName, string $routingKey)`- Registering binding between exchange and queue
4. Provides Consumer that will be registered at given name `"orders_consumer"` and will be polling `"orders"` queue

## Available Exchange configurations

```php
$amqpExchange = AmqpExchange::createDirectExchange
$amqpExchange = AmqpExchange::createFanoutExchange
$amqpExchange = AmqpExchange::createTopicExchange
$amqpExchange = AmqpExchange::createHeadersExchange

$amqpExchange = $amqpExchange
    ->withDurability(true) // exchanges survive broker restart
    ->withAutoDeletion() // exchange is deleted when last queue is unbound from it
```

## Available Queue configurations

```php
$amqpQueue = AmqpQueue::createDirectExchange
                ->withDurability(true) // the queue will survive a broker restart
                ->withExclusivity() // used by only one connection and the queue will be deleted when that connection closes
                ->withAutoDeletion() // queue that has had at least one consumer is deleted when last consumer unsubscribes
                ->withDeadLetterExchangeTarget($amqpExchange);
```

## Publisher Transactions

`Ecotone AMQP` comes with support for RabbitMQ Transaction for published messages. \
This means that, if you send more than one message at time, it will be commited together.

If you want to enable/disable for all [Asynchronous Endpoints](../tutorial-php-ddd-cqrs-event-sourcing/php-asynchronous-processing.md) or specific for Command Bus. You may use of `ServiceContext.`&#x20;

{% hint style="info" %}
By default RabbitMQ transactions are disabled, as you may ensure consistency using [Resilient Sending](../modelling/recovering-tracing-and-monitoring/resiliency/resilient-sending.md).
{% endhint %}

```php
class ChannelConfiguration
{
    #[ServiceContext]
    public function registerTransactions() : array
    {
        return [
            AmqpConfiguration::createWithDefaults()
                ->withTransactionOnAsynchronousEndpoints(false)
                ->withTransactionOnCommandBus(false)
        ];
    }

}
```

To enable transactions on specific endpoint if default is disabled, mark consumer with `Ecotone\Amqp\AmqpTransaction\AmqpTransaction` annotation.

```php
    #[AmqpTransaction)]
    #[MessageConsumer("consumer")]
    public function execute(string $message) : void
    {
        // do something with Message
    }
```
