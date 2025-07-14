# Message Publisher

## AMQP Distributed Bus

AMQP Distributed Bus is described in more details under [Distributed Bus section](../../modelling/microservices-php/distributed-bus/amqp-distributed-bus-rabbitmq/).

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

## Publisher Acknowledgments

By default Ecotone will aim for resiliency to avoid Message being lost. This protects from lost heartbeats issue (AMQP bug which make message vanish without exceptions) and ensures that Message are considered delivered only when Broker has acknowledged storing them on the Broker side (Using [Publisher confirms](https://www.rabbitmq.com/docs/confirms#publisher-confirms)).\
\
However Publisher confirms comes with time cost, as it makes publishing process awaits for acknowledge from RabbitMQ. Therefore if delivery guarantee is not an issue, and we can accept risk of losing messages we can consider disable it to speed up publishing time:

```php
#[ServiceContext]
public function amqpChannel() : array
{
    return [
        AmqpBackedMessageChannelBuilder::create("orders")
            ->withPublisherAcknowledgments(false)
    ];
}
```

{% hint style="success" %}
Publisher acknowledgments can be combined with [Outbox](../../modelling/recovering-tracing-and-monitoring/resiliency/outbox-pattern.md) to ensure high message guarantee.&#x20;
{% endhint %}

## Publisher Transactions

`Ecotone AMQP` comes with support for RabbitMQ Transaction for published messages. \
This means that, if you send more than one message at time, it will be commited together.

If you want to enable/disable for all [Asynchronous Endpoints](../../tutorial-php-ddd-cqrs-event-sourcing/php-asynchronous-processing.md) or specific for Command Bus. You may use of `ServiceContext.`&#x20;

{% hint style="info" %}
By default RabbitMQ transactions are disabled, as you may ensure consistency using [Resilient Sending](../../modelling/recovering-tracing-and-monitoring/resiliency/resilient-sending.md).
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
