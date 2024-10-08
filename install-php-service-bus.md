---
description: Installing Ecotone for Symfony, Laravel or Stand Alone
---

# Installation

## Install for Symfony

Use composer in order to download **Ecotone Symfony Bundle**

{% hint style="success" %}
composer require [ecotone/](https://packagist.org/packages/ecotone/)symfony-bundle
{% endhint %}

If you're using **Symfony Flex**,  bundle will auto-configure. \
If that did not happen, register bundle in **config/bundles.php**

```php
Ecotone\SymfonyBundle\EcotoneSymfonyBundle::class => ['all' => true]
```

{% hint style="warning" %}
By default Ecotone will look for Attributes in default Symfony catalog **"src"**. \
If you do follow different structure, you can use [**"namespaces"**](modules/symfony/symfony-ddd-cqrs-event-sourcing.md#namespaces) configuration to tell Ecotone, where to look for.&#x20;
{% endhint %}

***

## Install for Laravel

Use composer in order to download **Ecotone Laravel**

{% hint style="success" %}
composer require [ecotone/](https://packagist.org/packages/ecotone/)laravel
{% endhint %}

Provider should be automatically registered.\
If that did not happen, register provider

```php
'providers' => [
    \Ecotone\Laravel\EcotoneProvider::class
],
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
In order to start, you need have to `composer.json` with PSR-4 or PSR-0 autoload setup.
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
