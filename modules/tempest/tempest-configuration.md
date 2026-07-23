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

## Environment and Caching

Ecotone resolves the application environment from the `APP_ENV` variable, falling back to Tempest's own `ENVIRONMENT` convention. When the environment is `prod` or `production` (or when neither variable is visible to PHP), Ecotone uses the production cache: the compiled messaging system is stored once and trusted on every boot — no file scanning on the hot path.

The cache lives in `sys_get_temp_dir()/ecotone_tempest`.

{% hint style="warning" %}
Like Symfony's and Laravel's compiled containers, the production cache is trusted until cleared. Make `./tempest ecotone:cache:clear` part of your deployment, so code and configuration changes take effect.
{% endhint %}

Outside of production mode, the cache is keyed by your configuration and code, so changes are picked up automatically during development. If a stale production cache ever references a class that no longer exists, the error names the class and the cache directory, together with the fix.

## Console commands

The Ecotone commands are registered with Tempest's console and available through the `./tempest` executable:

```bash
# List all registered Ecotone endpoints
./tempest ecotone:list

# Run an asynchronous message consumer
./tempest ecotone:run <consumerName>

# Worker options for production cycles
./tempest ecotone:run <consumerName> --handledMessageLimit=100 --executionTimeLimit=30000 --memoryLimit=256

# Clear the Ecotone cache (e.g. after a deploy)
./tempest ecotone:cache:clear

# Inspect and replay failed messages (with the Dead Letter configured)
./tempest ecotone:deadletter:list
./tempest ecotone:deadletter:show <messageId>
./tempest ecotone:deadletter:replay <messageId>
./tempest ecotone:deadletter:replayAll
./tempest ecotone:deadletter:delete <messageId>
```

See [Asynchronous Processing and Workers](asynchronous-tempest.md) for worker lifecycle and dead-letter configuration.
