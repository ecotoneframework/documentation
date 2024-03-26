---
description: Ecotone support for Amazon SQS PHP
---

# Amazon SQS Support

## Installation

```php
composer require ecotone/sqs
```

### Module Powered By

[Enqueue](https://github.com/php-enqueue/enqueue-dev) solid and powerful abstraction over asynchronous queues.

## Configuration

In order to use `SQS Support` we need to add `ConnectionFactory` to our `Dependency Container.`&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
# config/services.yaml
Enqueue\Sqs\SqsConnectionFactory:
    class: Enqueue\Sqs\SqsConnectionFactory
    arguments:
        - "sqs:?key=key&secret=secret&region=us-east-1&version=latest"
```
{% endtab %}

{% tab title="Laravel" %}
```php
use Enqueue\Sqs\SqsConnectionFactory;

public function register()
{
     $this->app->singleton(SqsConnectionFactory::class, function () {
         return new SqsConnectionFactory("sqs:?key=key&secret=secret&region=us-east-1&version=latest");
     });
}
```
{% endtab %}

{% tab title="Lite" %}
```php
use Enqueue\Sqs\SqsConnectionFactory;

$application = EcotoneLiteApplication::boostrap(
    [
        SqsConnectionFactory::class => new SqsConnectionFactory("sqs:?key=key&secret=secret&region=us-east-1&version=latest")
    ]
);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
We register our `SqsConnectionFactory` under the class name `Enqueue\Sqs\SqsConnectionFactory.` This will help Ecotone resolve it automatically, without any additional configuration.
{% endhint %}

## Message Channel

To create [Message Channel](../modelling/asynchronous-handling/), we need to create [Service Context](../messaging/service-application-configuration.md).&#x20;

```php
use Ecotone\Sqs\SqsBackedMessageChannelBuilder;

class MessagingConfiguration
{
    #[ServiceContext] 
    public function orderChannel()
    {
        return SqsBackedMessageChannelBuilder::create("orders");
    }
}
```

Now `orders` channel will be available in Messaging System.&#x20;

### Message Channel Configuration

```php
SqsBackedMessageChannelBuilder::create("orders")
    ->withAutoDeclare(false) // do not auto declare queue
    ->withDefaultTimeToLive(1000) // limit TTL of messages
```

## Message Publisher

If you want to publish Message directly to Exchange, you may use of `Publisher.`

```php
use Ecotone\Sqs\Configuration\SqsMessagePublisherConfiguration;

class PublisherConfiguration
{
    #[ServiceContext] 
    public function registerPublisherConfig()
    {
        return 
            SqsMessagePublisherConfiguration::create(
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
SqsMessagePublisherConfiguration::create(queueName: $queueName)
    ->withAutoDeclareQueueOnSend(false) // 1
    ->withHeaderMapper("application.*") // 2
    
```

1. `withAutoDeclareQueueOnSend` - should Ecotone try to declare queue before sending message
2. `withHeaderMapper` - On default headers are not send with message. You map provide mapping for headers that should be mapped to `SQS Message`

## Message Consumer

To connect consumer directly to a SQS Queue, we need to provide `Ecotone` with information, how the Queue is configured.&#x20;

```php
use Ecotone\Sqs\Configuration\SqsMessageConsumerConfiguration;

class ConsumerConfiguration
{
    #[ServiceContext] 
    public function registerConsumerConfig(): array
    {
        return [
            SqsMessageConsumerConfiguration::create("orders_consumer", "orders")
        ];
    }
}
```

1. Provides Consumer that will be registered at given name `"orders_consumer"` and will be polling `"orders"` queue

### Consumer Configuration

```php
$consumerConfiguration = SqsMessageConsumerConfiguration::createDirectExchange()
                ->withDeclareOnStartup(false) // do not try to declare queue before consuming first message;
```
