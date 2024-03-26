---
description: Ecotone support for Redis
---

# Redis Support

## Installation

```php
composer require ecotone/redis
```

### Module Powered By

[Enqueue](https://github.com/php-enqueue/enqueue-dev) solid and powerful abstraction over asynchronous queues.

## Configuration

In order to use `Redis Support` we need to add `ConnectionFactory` to our `Dependency Container.`&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
# config/services.yaml
Enqueue\Redis\RedisConnectionFactory:
    class: Enqueue\Redis\RedisConnectionFactory
    arguments:
        - "redis://localhost:6379"
```
{% endtab %}

{% tab title="Laravel" %}
```php
use Enqueue\Redis\RedisConnectionFactory;

public function register()
{
     $this->app->singleton(RedisConnectionFactory::class, function () {
         return new RedisConnectionFactory("redis://localhost:6379");
     });
}
```
{% endtab %}

{% tab title="Lite" %}
```php
use Enqueue\Redis\RedisConnectionFactory;

$application = EcotoneLiteApplication::boostrap(
    [
        RedisConnectionFactory::class => new RedisConnectionFactory("redis://localhost:6379")
    ]
);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
We register our RedisConnectionFactory under the class name `Enqueue\Redis\`RedisConnectionFactory`.` This will help Ecotone resolve it automatically, without any additional configuration.
{% endhint %}

## Message Channel

To create [Message Channel](../modelling/asynchronous-handling/), we need to create [Service Context](../messaging/service-application-configuration.md).&#x20;

```php
use Ecotone\Redis\RedisBackedMessageChannelBuilder;

class MessagingConfiguration
{
    #[ServiceContext] 
    public function orderChannel()
    {
        return RedisBackedMessageChannelBuilder::create("orders");
    }
}
```

Now `orders` channel will be available in Messaging System.&#x20;

### Message Channel Configuration

```php
RedisBackedMessageChannelBuilder::create("orders")
    ->withAutoDeclare(false) // do not auto declare queue
    ->withDefaultTimeToLive(1000) // limit TTL of messages
```

## Message Publisher

If you want to publish Message directly to Exchange, you may use of `Publisher.`

```php
use Ecotone\Redis\Configuration\RedisMessageConsumerConfiguration;

class PublisherConfiguration
{
    #[ServiceContext] 
    public function registerPublisherConfig()
    {
        return 
            RedisMessagePublisherConfiguration::create(
                MessagePublisher::class, // 1
                "delivery", // 2
                "application/json" // 3
            );
    }
}
```

1. `Reference name` -  Name under which it will be available in Dependency Container.
2. `Queue name` - Name of queue where Message should be published
3. `Default Conversion [Optional]` - Default type, payload will be converted to.

### Publisher Configuration

```php
RedisMessagePublisherConfiguration::create(queueName: $queueName)
    ->withAutoDeclareQueueOnSend(false) // 1
    ->withHeaderMapper("application.*") // 2
    
```

1. `withAutoDeclareQueueOnSend` - should Ecotone try to declare queue before sending message
2. `withHeaderMapper` - On default headers are not send with message. You map provide mapping for headers that should be mapped to `Redis Message`

## Message Consumer

To connect consumer directly to a Redis Queue, we need to provide `Ecotone` with information, how the Queue is configured.&#x20;

```php
use Ecotone\Redis\Configuration\RedisMessageConsumerConfiguration;

class ConsumerConfiguration
{
    #[ServiceContext] 
    public function registerConsumerConfig(): array
    {
        return [
            RedisMessageConsumerConfiguration::create("orders_consumer", "orders")
        ];
    }
}
```

1. Provides Consumer that will be registered at given name `"orders_consumer"` and will be polling `"orders"` queue

### Consumer Configuration

```php
$consumerConfiguration = RedisMessageConsumerConfiguration::createDirectExchange
                ->withDeclareOnStartup(false) // do not try to declare queue before consuming first message;
```
