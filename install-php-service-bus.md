---
description: Installing Ecotone for Symfony, Laravel or Stand Alone
---

# Installation

## Prerequisites

Before installing Ecotone, ensure you have:
- PHP 8.1 or higher
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
