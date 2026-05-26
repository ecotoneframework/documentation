---
description: Event Sourcing DDD CQRS Symfony PHP
---

# Symfony Configuration

## Configuration

```php
ecotone: 
    loadSrcNamespaces: bool (default: true)
    failFast: bool (default: true, production: false)
    namespaces: string[] (default: [])
    defaultSerializationMediaType: string (default: application/x-php-serialized) [application/json, application/xml]
    defaultErrorChannel: string (default: null)
    defaultMemoryLimit: string (default: 1024)
    defaultConnectionExceptionRetry: 
       initialDelay: int (default: 100, production: 1000)
       maxAttempts: int (default: 3, production: 5)
       multiplier: int (default: 3)
    serviceName: string (default: null)
    skippedModulePackageNames: string[] (default: [])
    test: bool (default: false)
    licenceKey: string|null (default: null)
    
```

### loadSrcNamespaces

Tells Ecotone, if should automatically load all namespaces defined for `src` catalog

### failFast

Describes if Ecotone should fail fast. \
If `true`, then Ecotone will boot all endpoints during each request, so it can inform, if configuration is incorrect immediately, it provides fast feedback for the developer.\
if `false,` then Ecotone will not boot up any endpoints at each request, which will increase performance, but will results in slower feedback for the developer.

### namespaces

List of namespace prefixes, that Ecotone should look attributes for.

### defaultSerializationMediaType

Describes default serialization type within application. If not configured default serialization will be `application/x-php-serialized,`which is serialized PHP class.

### defaultErrorChannel

Provides default [Poller configuration](../../modelling/asynchronous-handling/scheduling.md#polling-metadata) with error channel for all [asynchronous consumers](../../messaging/messaging-concepts/consumer.md#polling-consumer).

### defaultMemoryLimit

Provides default memory limit in megabytes for all [asynchronous consumers](../../messaging/messaging-concepts/consumer.md#polling-consumer).

### defaultConnectionExceptionRetry

Provides default connection retry strategy for [asynchronous consumers](../../messaging/messaging-concepts/consumer.md#polling-consumer) in case of connection failure.&#x20;

`initialDelay` - delay after first retry in milliseconds\
`multiplier` - how much initialDelay should be multipled with each try\
`maxAttempts` - How many attemps should be done, before closing closing endpoint

### serviceName

If you're running distributed services (microservices) and want to use Ecotone's [capabilities for integration](../../modelling/microservices-php/), then provide name for the service (application).&#x20;

### skippedModulePackageNames

Skip list of given module package names (Check`ModulePackageList` for available packages).

### test

Should test mode be enabled, so `MessagingTestSupport` can be used.

### licenceKey

Provides access to Enterprise Feature of Ecotone.

## Using Environment Variables in Attributes

Ecotone attributes are compiled into the Symfony container, so you can reference Symfony's `%env(...)%` placeholders directly inside attribute arguments. Ecotone keeps the value as a regular container argument, and Symfony resolves it from the environment — just like any other service argument.

This is useful when a value such as a Kafka topic or an error channel name should differ per environment.

```php
use Ecotone\Kafka\Attribute\KafkaConsumer;

final class OrderConsumer
{
    #[KafkaConsumer(
        endpointId: 'orderConsumer',
        topics: ['orders.%env(KAFKA_TOPIC_SUFFIX)%']
    )]
    public function handle(string $payload): void
    {
        // KAFKA_TOPIC_SUFFIX=production -> consumes from "orders.production"
    }
}
```

The same applies to any other attribute argument, for example the error channel name:

```php
use Ecotone\Kafka\Attribute\KafkaConsumer;
use Ecotone\Messaging\Attribute\ErrorChannel;

final class OrderConsumer
{
    #[ErrorChannel('errors.%env(ORDER_ERROR_CHANNEL)%')]
    #[KafkaConsumer(
        endpointId: 'orderConsumer',
        topics: ['orders']
    )]
    public function handle(string $payload): void
    {
        // ORDER_ERROR_CHANNEL=high_priority -> failures routed to "errors.high_priority"
    }
}
```

Declare the variables as you would any other Symfony environment variable:

```bash
# .env
KAFKA_TOPIC_SUFFIX=production
ORDER_ERROR_CHANNEL=high_priority
```

{% hint style="info" %}
The `%env(...)%` placeholder needs to be present when the container is compiled; its value is read from the environment at runtime, so changing the value does not require clearing the cache.
{% endhint %}
