# Ecotone Lite

Ecotone Lite is standalone solution for running Ecotone Framework. You may combine it with your internal company framework, use it alone or combine it with any other framework (e.g. Laminas, CodeIgniter, Magento).\
Ecotone Lite is also a [great way to run your tests](../../modelling/testing-support/testing-messaging.md).

## Installation

Follow Installation from [Installation menu](../../install-php-service-bus.md#install-lite-no-framework).

{% hint style="success" %}
In order for Ecotone to look for attributes in given catalog, provide ["withLoadCatalog"](./#withloadcatalog) configuration. You may also provide list of [namespaces](./#withnamespaces) instead.
{% endhint %}

## Configuration

```php
$messagingSystem = EcotoneLite::bootstrap(
    $listOfClassesToResolve,
    $containerOrListOfClassInstances,
    ServiceConfiguration::createWithDefaults()
        ->withEnvironment("prod")
        ->withLoadCatalog("src")
        ->withFailFast(false)
        ->withNamespaces(["App"])
        ->withDefaultSerializationMediaType("application/json")                
        ->withDefaultErrorChannel("errorChannel")
        ->withConsumerMemoryLimit(512)
        ->withCacheDirectoryPath("/var/www/cache")
        ->withConnectionRetryTemplate(RetryTemplateBuilder::fixedBackOff(100))
        ->withServiceName("banking")
        ->withSkippedModulePackageNames(ModulePackageList::allPackages())
        ->withExtensionObjects([InMemoryRepositoryBuilder::createForAllStateStoredAggregates()]),
        ->withLicenceKey('ecotoneEnterpriseKey')
    $configurationVariables,
    $useCachedVersion
);
    
```

* `$listOfClassesToResolve` - List of classes, which are not included in your                                 `->withNamespaces` or are not under `->withLoadCatalog` that you would like to load.
* `$containerOrListOfClassInstance` - PSR compatible container or list of class instance used in your Application.
* `ServiceConfiguration` - Provides extra configuration for you Ecotone Application
* `$configurationVariables` - Associative array of your configuration variables (like database connection string)
* `$useCachedVersion` - Should Ecotone use cached version of Ecotone Lite, if available

## ServiceConfiguration

### withEnvironment()

Tells Ecotone what kind of environment type is currently running

### withLoadCatalog()

Tells Ecotone, if to automatically load all namespaces defined for given catalog.

### withFailFast()

Describes if Ecotone should fail fast. \
If `true`, then Ecotone will boot all endpoints during each request, so it can inform, if configuration is incorrect immediately, it provides fast feedback for the developer.\
if `false,` then Ecotone will not boot up any endpoints at each request, which will increase performance, but will results in slower feedback for the developer.

### withNamespaces()

List of namespace prefixes, that Ecotone should look attributes for.

### withDefaultSerializationMediaType()

Describes default serialization type within application. If not configured default serialization will be `application/x-php-serialized,`which is serialized PHP class.

### withDefaultErrorChannel()

Provides default [Poller configuration](../../modelling/asynchronous-handling/scheduling.md#polling-metadata) with error channel for all [asynchronous consumers](../../messaging/messaging-concepts/consumer.md#polling-consumer).

### withDefaultMemoryLimit()

Provides default memory limit in megabytes for all [asynchronous consumers](../../messaging/messaging-concepts/consumer.md#polling-consumer).

### withDefaultConnectionExceptionRetry()

Provides default connection retry strategy for [asynchronous consumers](../../messaging/messaging-concepts/consumer.md#polling-consumer) in case of connection failure.&#x20;

`initialDelay` - delay after first retry in milliseconds\
`multiplier` - how much initialDelay should be multipled with each try\
`maxAttempts` - How many attemps should be done, before closing closing endpoint

### serviceName()

If you're running distributed services (microservices) and want to use Ecotone's [capabilities for integration](../../modelling/microservices-php/), then provide name for the service (application).&#x20;

### withSkippedModulePackageNames()

Skip list of given module package names (Check`ModulePackageList` for available packages)

### withExtensionObjects()

Provides list of extension object for your Ecotone's modules, that adjust the behaviour to your needs.

### withLicenceKey()

Provides access to Enterprise Feature of Ecotone.
