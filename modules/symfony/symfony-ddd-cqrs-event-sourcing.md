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
