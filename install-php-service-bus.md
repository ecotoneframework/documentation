---
description: Installing Ecotone for Symfony, Laravel or Stand Alone
---

# Installation

## Install for Symfony

1. Use composer in order to download `Ecotone Symfony Bundle`

{% hint style="success" %}
composer require [ecotone/](https://packagist.org/packages/ecotone/)symfony-bundle
{% endhint %}

If you're using _`Symfony Flex`_,  bundle will auto-configure.&#x20;

2\. Register bundle, if needed

{% hint style="success" %}
new Ecotone\SymfonyBundle\EcotoneSymfonyBundle::class => \['all' => true]
{% endhint %}

## Install for Laravel

1. Use composer in order to download `Ecotone Laravel`

{% hint style="success" %}
composer require [ecotone/](https://packagist.org/packages/ecotone/)laravel
{% endhint %}

Provider should be automatically registered.

2\. Register provider, if needed

```php
'providers' => [
    \Ecotone\Laravel\EcotoneProvider::class
],
```

## Install Ecotone Lite (No framework)

If you're using no framework or framework `different` than `Symfony` or `Laravel`, then you may use `Ecotone Lite` to bootstrap Ecotone.

{% hint style="info" %}
In order to start, you need have to `composer.json` with PSR-4 or PSR-0 autoload setup.
{% endhint %}

### Ecotone Lite Application

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
With default configuration, Ecotone will look for classes inside `src` catalog.
{% endhint %}

### With Custom Dependency Container

If you already have Dependency Container configured, then:

{% hint style="success" %}
composer require ecotone/ecotone
{% endhint %}

```php
$ecotoneLite = EcotoneLite::bootstrap(
    containerOrAvailableServices: $container
);
```

### With no Dependency Container

You may actually run Ecotone without any Dependency Container. That may be useful for small applications, testing or when we want just run some Ecotone's script.

{% hint style="success" %}
composer require ecotone/ecotone
{% endhint %}

```php
$ecotoneLite = EcotoneLite::bootstrap(
    [User::class, UserRepository::class, UserService::class],
    [new UserRepository(), new UserService()]
);
```
