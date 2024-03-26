# Logging

## Logging with Ecotone Lite

In order to setup logs for Ecotone Lite, we will need to pass [PSR compatible Logger](https://github.com/php-fig/log) to known Services.

```php
$ecotoneLite = EcotoneLite::bootstrap(
    [OrderService::class],
    containerOrAvailableServices: [
        // pass your Logger here
        'logger' => $loggerService,
    ]
)
```

{% hint style="success" %}
If you pass Dependency Container, then the container should provide the Logger under **"logger"** reference.
{% endhint %}
