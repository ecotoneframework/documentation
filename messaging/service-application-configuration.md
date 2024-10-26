---
description: Ecotone Framework customization
---

# Service (Application) Configuration

`Ecotone` allows for customization of the core functionality as well as the [modules](../modules/overview.md).

## Ecotone Core Configuration

Depending on the provider you choose, you can follow configuration in specific menu

1. [Symfony Configuration](../modules/symfony/symfony-ddd-cqrs-event-sourcing.md#configuration)
2. [Laravel Configuration](../modules/laravel/laravel-ddd-cqrs-event-sourcing.md#configuration)
3. [Ecotone Lite](../modules/ecotone-lite/#configuration)

## Module Extensions

Module Extensions are configurations for specific module. It's class based configuration.\
\
Let's take a look on [Dbal](../modules/dbal-support.md) module configuration as example:&#x20;

```php
class MyConfiguration // 1
{
    #[ServiceContext] // 2
    public function registerTransactions() // 3
    {
        return DbalConfiguration::createWithDefaults() // 4
                ->withTransactionOnAsynchronousEndpoints(true)
                ->withTransactionOnCommandBus(true);
    }
}
```

1. Create your own class. You can name it whatever you like.
2. Add attribute to let `Ecotone` know that it should call this method to get the configuration.
3. Name the method whatever you like. You may return `array` of configurations or `specific configuration instance`.
4. Return specific configuration.&#x20;

{% hint style="info" %}
Ecotone does not require specific class name or method name. \
All what is needed is `#[ServiceContext]` attribute.
{% endhint %}

### Environment Specific Configuration

If you want to enable different configuration for specific environment you may use of `Environment attribute`.

```php
class MyConfiguration
{
    #[ServiceContext]
    #[Environment(["dev", "prod"])]
    public function registerTransactions()
    {
        return DbalConfiguration::createWithDefaults()
                ->withTransactionOnAsynchronousEndpoints(true)
                ->withTransactionOnCommandBus(true);
    }
    
    #[ServiceContext]
    #[Environment(["test"])]
    public function registerTransactions()
    {
        return DbalConfiguration::createWithDefaults()
                ->withTransactionOnAsynchronousEndpoints(false)
                ->withTransactionOnCommandBus(false);
    }
}
```

Above will turn off transactions for `test` environment, keeping it however for `prod` and `dev`.

### Configuration Variables

You may access your configuration variables inside `ServiceContext` methods.

```php
class MyConfiguration
{
    #[ServiceContext]
    public function registerTransactions(#[ConfigurationVariable("database")] string $connectionDsn)
    {
        return Repository($connectionDsn);
    }
}
```

If you don't pass `ConfigurationVariable` attribute, it will be taken from parameter name. \
Below example is equal to above.

```php
class MyConfiguration
{
    #[ServiceContext]
    public function registerTransactions(string $database)
    {
        return Repository($database);
    }
}
```

{% hint style="warning" %}
Service Context is evaluated before container is dumped and cached. Therefore if you will change environment variables after your cache is dumped this won't be changed.\
\
This happens because Ecotone tries to maximalize configuration caching, in order to speed up run time execution and do no configuration at that time.
{% endhint %}
