---
description: Type-safe Closures as alternative to expression language in PHP attributes
---

# Closures as Expressions

PHP 8.5 introduced [Closures in constant expressions](https://www.php.net/releases/8.5/en.php), which makes it possible to define Closures directly inside Attributes. Ecotone builds on top of that: **every Attribute that provides an `expression` property accepts a Closure as an alternative to the** [**Symfony Expression Language**](https://symfony.com/doc/current/components/expression_language.html) **string**.

```php
// Expression Language
#[Delayed(expression: 'payload.dueDate')]

// Closure
#[Delayed(expression: static function (#[Payload] OrderWasPlaced $event): \DateTimeInterface {
    return $event->dueDate;
})]
```

While Expression Language strings are compact, Closures give you:

* **Type safety and static analysis** - PHPStan/Psalm verify the code, misspelled properties fail before deployment, not in production
* **IDE support** - autocompletion, go-to-definition and find-usages just work
* **Refactoring safety** - renaming a property or method updates the Closure automatically
* **No new syntax to learn** - it is plain PHP instead of a separate expression dialect

{% hint style="success" %}
Closures as Expressions are available as part of **Ecotone Enterprise** and require **PHP 8.5+**.\
Without an Enterprise licence, using a Closure expression fails fast during application bootstrap with a `LicensingException`.
{% endhint %}

## How Closure parameters are resolved

The Closure is executed like a regular [Message Handler](method-invocation.md) - its parameters are resolved from the currently handled Message using the same Parameter Converters you already know:

```php
#[CommandHandler('order.place')]
public function placeOrder(
    PlaceOrder $command,
    #[Header('token', expression: static function (
        #[Header('token')] string $token,
        #[Reference] TokenService $tokenService,
    ): string {
        return $tokenService->normalize($token);
    })] string $token,
): void {}
```

Inside the Closure you may use:

* `#[Payload]` - payload of the Message (with [Conversion](conversion/) applied, e.g. deserialized Command)
* `#[Header('name')]` / `#[Headers]` - specific header or all headers
* `#[Reference]` - Service from Dependency Container
* `#[ConfigurationVariable]` - configuration parameter
* `Message` type hint - the whole Message
* [Default Converters](method-invocation.md#default-converters) - first parameter defaults to payload, class type hints resolve Services from the container

All of them can be combined within single Closure:

```php
#[AddHeader('shipping.method', expression: static function (
    #[Payload] ShipOrder $command,                                      // deserialized Message payload
    #[Header('region')] string $region,                                 // single Message header
    #[Headers] array $metadata,                                         // all Message headers
    #[Reference] ShippingResolver $resolver,                            // Service from Dependency Container
    #[ConfigurationVariable('defaultCarrier')] string $defaultCarrier,  // configuration parameter
): string {
    return $resolver->resolve($command, $region, $metadata) ?? $defaultCarrier;
})]
#[Asynchronous('orders')]
#[CommandHandler(endpointId: 'shipOrderEndpoint')]
public function ship(ShipOrder $command): void {}
```

And thanks to Default Converters, simple cases need no attributes at all - the first parameter receives the payload and class type hints resolve Services:

```php
#[Delayed(expression: static function (OrderWasPlaced $event, DelayPolicy $delayPolicy): int {
    return $delayPolicy->delayInMillisecondsFor($event);
})]
```

### Context variables bound by parameter name

Expression Language exposes special variables like `value` or `service`. For Closures, those are bound **by parameter name**, so you can keep the same semantics without extra attributes:

* `$value` in `#[Header]` - receives the given header value
* `$value` in `#[Payload]` and `#[Fetch]` - receives the Message payload
* `$service` in `#[Reference]` - receives the referenced Service

```php
// `$value` receives the `token` header - same as `value` in Expression Language
#[Header('token', expression: static function (string $value): string {
    return strrev($value);
})] string $token

// `$service` receives the referenced Service
#[Reference('globalConfigurationService', expression: static function (object $service): LocalConfiguration {
    return $service->getCurrentConfiguration();
})] LocalConfiguration $localConfiguration
```

For [Database Business Methods](#database-business-methods) there is no Message involved, so Closure parameters bind by name to the Business Method parameters instead.

## Use cases

### Deriving Message Handler parameters

Transform payload or headers before they reach your handler - [Method Invocation](method-invocation.md):

```php
#[CommandHandler('order.total')]
public function calculateTotal(
    #[Payload(expression: static function (#[Payload] array $order, #[Headers] array $headers): int {
        return array_sum($order['items']) + ($headers['fee'] ?? 0);
    })] int $total,
): void {}
```

### Delaying Messages and Time to Live

Compute delay or expiration from the Message itself - [Delaying Messages](../../modelling/asynchronous-handling/delaying-messages.md), [Time to Live](../../modelling/asynchronous-handling/time-to-live.md):

```php
#[Delayed(expression: static function (#[Payload] NotifyCustomer $command): int {
    return $command->delayInMilliseconds;
})]
#[Asynchronous('notifications')]
#[CommandHandler('customer.notify', endpointId: 'customerNotifyEndpoint')]
public function notify(NotifyCustomer $command): void {}
```

### Enriching Message Headers

Add dynamically computed headers when Message is sent to asynchronous channel:

```php
#[AddHeader('priority.level', expression: static function (#[Payload] PlaceOrder $command): string {
    return $command->isVip ? 'high' : 'normal';
})]
#[Asynchronous('orders')]
#[CommandHandler(endpointId: 'placeOrderEndpoint')]
public function placeOrder(PlaceOrder $command): void {}
```

### Fetching Aggregates

Resolve Aggregate identifiers with full type safety - [Instant Fetch Aggregate](../../modelling/command-handling/repository/repository.md#instant-fetch-aggregate):

```php
#[CommandHandler]
public function placeOrder(
    PlaceOrder $command,
    #[Fetch(static function (#[Payload] PlaceOrder $command): string {
        return $command->userId;
    })] User $user,
): void {}
```

### Deduplication

Derive deduplication keys from Message data - [Idempotent Consumer](../../modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md):

```php
#[Deduplicated(expression: static function (#[Header('orderId')] string $orderId): string {
    return $orderId;
})]
#[CommandHandler('order.place', endpointId: 'placeOrderEndpoint')]
public function placeOrder(PlaceOrder $command): void {}
```

### Tenant resolution

Derive the tenant from inbound Messages - [Multi-Tenancy](../multi-tenancy-support/different-scenarios/deriving-tenant-from-inbound-messages.md):

```php
#[Scheduled(requestChannelName: 'externalEventArrived', endpointId: 'externalEventPoller')]
#[WithTenantResolver(expression: static function (#[Header('source')] string $source): string {
    return $source;
})]
public function poll(): ?Message {}
```

### Database Business Methods

For [Dbal Parameters](../../modelling/command-handling/business-interface/working-with-database/converting-parameters.md) there is no Message at evaluation time, therefore Closure parameters are bound **by name to the Business Method parameters** (and `$payload` for parameter level `DbalParameter`):

```php
#[DbalWrite('INSERT INTO persons (person_id, name) VALUES (:personId, :fullName)')]
#[DbalParameter('fullName', expression: static function (string $firstName, string $lastName): string {
    return $firstName . ' ' . $lastName;
})]
public function insert(int $personId, string $firstName, string $lastName): void;
```

## Good to know

* **Closures must be static and self-contained** - PHP requires Closures inside Attributes to be declared `static` and they cannot capture variables (`use`) or `$this`
* **No nested Closure expressions** - parameters of a Closure expression may use Parameter Converters with Expression Language strings (e.g. `#[Header('name', expression: 'value.trim()')]`), but not another Closure expression
* **Works with compiled containers** - Ecotone resolves the Closure's parameter converters during container compilation and re-reads the Closure from its Attribute declaration at runtime, so production setups with dumped container cache are fully supported
* **Closures in your own custom Attributes** (without `expression` semantics) are supported out of the box in Ecotone Free - the Enterprise licence applies to `expression` properties of Ecotone's Attributes
