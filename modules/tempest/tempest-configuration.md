---
description: Event Sourcing DDD CQRS Tempest
---

# Tempest Configuration

Ecotone works with zero configuration in a Tempest application — handlers are auto-discovered from your application's PSR-4 roots. To customise the behaviour, create a discovered `ecotone.config.php` returning an `EcotoneConfig` object:

```php
use Ecotone\Tempest\EcotoneConfig;

return new EcotoneConfig(
    serviceName: 'orders',
    licenceKey: env('ECOTONE_LICENCE_KEY'),
);
```

## Configuration

```php
loadAppNamespaces: bool (default: true)
cacheConfiguration: bool (default: false, production: true)
namespaces: string[] (default: [])
defaultSerializationMediaType: string (default: application/x-php-serialized) [application/json, application/xml]
defaultErrorChannel: string (default: null)
serviceName: string (default: null)
skippedModulePackageNames: string[] (default: [])
test: bool (default: false)
licenceKey: string|null (default: null)
```

### loadAppNamespaces

When `true` (default), Ecotone derives the namespaces to scan for Attributes from your Tempest application's PSR-4 roots (for example `App\`). This is the Tempest equivalent of scanning Laravel's `app` or Symfony's `src` catalog.

### cacheConfiguration

Describes if Ecotone should cache configuration.\
If `true`, then Ecotone will cache all configuration — this increases application load time but results in slower feedback for the developer as the cache needs to be removed on change.\
If `false`, then Ecotone will not cache configuration — this decreases application load time but results in quicker feedback for the developer.

### namespaces

List of namespace prefixes that Ecotone should look for Attributes in. When provided, this overrides the namespaces derived from `loadAppNamespaces`.

### defaultSerializationMediaType

Describes the default serialization type within the application. If not configured the default serialization will be `application/x-php-serialized`, which is a serialized PHP class.

### defaultErrorChannel

Provides default [Poller configuration](../../modelling/asynchronous-handling/scheduling.md#polling-metadata) with an error channel for all [asynchronous consumers](../../messaging/messaging-concepts/consumer.md#polling-consumer).

### serviceName

If you're running distributed services (microservices) and want to use Ecotone's [capabilities for integration](../../modelling/microservices-php/), then provide a name for the service (application).

### skippedModulePackageNames

Skip list of given module package names (check `ModulePackageList` for available packages).

### test

Should test mode be enabled, so `MessagingTestSupport` can be used.

### licenceKey

Provides access to Enterprise Features of Ecotone.

## Console commands

The Ecotone commands are registered with Tempest's console and available through the `./tempest` executable:

```bash
# List all registered Ecotone endpoints
./tempest ecotone:list

# Run an asynchronous message consumer
./tempest ecotone:run <consumerName>

# Clear the Ecotone cache (e.g. after a deploy)
./tempest ecotone:cache:clear
```
