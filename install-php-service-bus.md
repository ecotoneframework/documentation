---
description: Installing Ecotone for Symfony, Laravel or Stand Alone
---

# Installation

Ecotone is the PHP architecture layer that grows with your system without rewrites. One Composer package adds CQRS, event sourcing, sagas, projections, outbox, EIP routing, and distributed messaging to Laravel or Symfony via declarative PHP attributes — no rewrite, no bespoke glue.

Pick your stack below. Symfony and Laravel get dedicated integration packages with auto-configuration; any other framework (or no framework) runs on **Ecotone Lite** through a PSR-11 container.

## Prerequisites

Before installing Ecotone, ensure you have:
- PHP 8.2 or higher
- Composer installed
- A properly configured PHP project with PSR-4 autoloading

## Install for Symfony

**Step 1:** Install the Ecotone Symfony Bundle using Composer

{% hint style="success" %}
composer require [ecotone/](https://packagist.org/packages/ecotone/)symfony-bundle
{% endhint %}

**Step 2:** Verify Bundle Registration

If you're using **Symfony Flex** (recommended), the bundle will auto-configure. \
If auto-configuration didn't work, manually register the bundle in **config/bundles.php**:

```php
Ecotone\SymfonyBundle\EcotoneSymfonyBundle::class => ['all' => true]
```

**Step 3:** Verify Installation

Run this command to check if Ecotone is properly installed:

```bash
php bin/console ecotone:list
```

{% hint style="warning" %}
By default Ecotone will look for Attributes in default Symfony catalog **"src"**. \
If you do follow different structure, you can use [**"namespaces"**](modules/symfony/symfony-ddd-cqrs-event-sourcing.md#namespaces) configuration to tell Ecotone, where to look for.&#x20;
{% endhint %}

{% hint style="info" %}
**Next steps for Symfony:** reuse a Connection already defined in your application ([Doctrine DBAL connection](modules/symfony/symfony-database-connection-dbal-module.md)), turn an existing Doctrine entity into an [Aggregate](modules/symfony/doctrine-orm.md), or run handlers asynchronously on an existing [Messenger transport](modules/symfony/symfony-messenger-transport.md).
{% endhint %}

***

## Install for Laravel

**Step 1:** Install the Ecotone Laravel Package

{% hint style="success" %}
composer require [ecotone/](https://packagist.org/packages/ecotone/)laravel
{% endhint %}

**Step 2:** Verify Provider Registration

The service provider should be automatically registered via Laravel's package discovery.\
If auto-registration didn't work, manually add the provider to **config/app.php**:

```php
'providers' => [
    \Ecotone\Laravel\EcotoneProvider::class
],
```

**Step 3:** Verify Installation

Run this command to check if Ecotone is properly installed:

```bash
php artisan ecotone:list
```

{% hint style="warning" %}
By default Ecotone will look for Attributes in default Laravel catalog **"app"**. \
If you do follow different structure, you can use [**"namespaces"**](modules/laravel/laravel-ddd-cqrs-event-sourcing.md#namespaces) configuration to tell Ecotone, where to look for.&#x20;
{% endhint %}

**Step 4 (optional):** Publish the configuration file

```bash
php artisan vendor:publish --tag=ecotone-config
```

This creates `config/ecotone.php`, where you can configure namespaces, cache, serialization, the error channel, and your Enterprise licence key.

{% hint style="info" %}
**Next steps for Laravel:** reuse a connection from `config/database.php` ([Doctrine DBAL connection](modules/laravel/database-connection-dbal-module.md)), use an [Eloquent model as an Aggregate](modules/laravel/eloquent.md), or run handlers asynchronously on your existing [Laravel queues](modules/laravel/laravel-queues.md).
{% endhint %}

***

## Install Ecotone Lite (No framework)

If you're using no framework or framework **different** than **Symfony** or **Laravel**, then you may use **Ecotone Lite** to bootstrap Ecotone.

{% hint style="success" %}
composer require ecotone/ecotone
{% endhint %}

{% hint style="info" %}
In order to start, you need to have a `composer.json` with PSR-4 or PSR-0 autoload setup.
{% endhint %}

### With Custom Dependency Container

If you already have Dependency Container configured, then:

```php
$ecotoneLite = EcotoneLite::bootstrap(
    classesToResolve: [User::class, UserRepository::class, UserService::class],
    containerOrAvailableServices: $container
);
```

### Load namespaces

By default Ecotone will look for Attributes only in Classes provided under **"classesToResolve"**. \
If we want to look for Attributes in given set of Namespaces, we can pass it to the configuration.

```php
$ecotoneLite = EcotoneLite::bootstrap(
    classesToResolve: [User::class, UserRepository::class, UserService::class],
    containerOrAvailableServices: $container,
    configuration: ServiceConfiguration::createWithDefaults()->withNamespaces(['App'])
);
```

### With no Dependency Container

You may actually run Ecotone without any Dependency Container. That may be useful for small applications, testing or when we want to run some small Ecotone's script.

```php
$ecotoneLite = EcotoneLite::bootstrap(
    classesToResolve: [User::class, UserRepository::class, UserService::class],
    containerOrAvailableServices: [new UserRepository(), new UserService()]
);
```

***

## Ecotone Lite Application

You may use out of the box Ecotone Lite Application, which provide you with Dependency Container.

{% hint style="success" %}
composer require ecotone/lite-application
{% endhint %}

```php
$ecotoneLite = EcotoneLiteApplication::bootstrap();

$commandBus = $ecotoneLite->getCommandBus();
$queryBus = $ecotoneLite->getQueryBus();
```

{% hint style="info" %}
With default configuration, Ecotone will look for classes inside **"src"** catalog.
{% endhint %}

## Database tables

Ecotone creates tables only for the features you actually use, and they live in **your** database — your backups, your retention, your access control:

| Feature | Default table |
| --- | --- |
| Transactional outbox / DBAL message channel | `enqueue` |
| Dead letter (stored failed messages) | `ecotone_error_messages` |
| Deduplication | `ecotone_deduplication` |
| Document store | `ecotone_document_store` |
| Event store (Event Sourcing) | per-stream tables |

By default these are created automatically on first use. To create them explicitly — or to ship them through your own migration tooling — use the console command:

```bash
# Symfony
php bin/console ecotone:migration:database:setup --initialize
# Laravel
php artisan ecotone:migration:database:setup --initialize
```

Pass `--sql` instead of `--initialize` to print the `CREATE TABLE` statements, so you can paste them into your own Doctrine Migrations or Laravel migration.

## Common Installation Issues

### "Class not found" errors
**Problem:** Ecotone can't find your classes with attributes.
**Solution:** Make sure your classes are in the correct namespace and directory structure matches your PSR-4 autoloading configuration.

### Bundle/Provider not registered
**Problem:** Ecotone commands are not available.
**Solution:**
- For Symfony: Check that the bundle is listed in `config/bundles.php`
- For Laravel: Check that the provider is in `config/app.php` or that package discovery is enabled

### Permission errors
**Problem:** Cache directory is not writable.
**Solution:** Ensure your web server has write permissions to the cache directory (usually `var/cache` for Symfony or `storage/framework/cache` for Laravel).
