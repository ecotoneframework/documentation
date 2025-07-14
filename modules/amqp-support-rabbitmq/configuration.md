# Configuration

## Installation

```php
composer require ecotone/amqp
```

### Module Powered By

[Enqueue](https://github.com/php-enqueue/enqueue-dev) solid and powerful abstraction over asynchronous queues.

## Configuration

In order to use `AMQP Support` we need to add `ConnectionFactory` to our `Dependency Container.`&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
# config/services.yaml
# You need to have RabbitMQ instance running on your localhost, or change DSN
    Enqueue\AmqpExt\AmqpConnectionFactory:
        class: Enqueue\AmqpExt\AmqpConnectionFactory
        arguments:
            - "amqp://guest:guest@localhost:5672//"
```
{% endtab %}

{% tab title="Laravel" %}
```php
# Register AMQP Service in Provider

use Enqueue\AmqpExt\AmqpConnectionFactory;

public function register()
{
     $this->app->singleton(AmqpConnectionFactory::class, function () {
         return new AmqpConnectionFactory("amqp+lib://guest:guest@localhost:5672//");
     });
}
```
{% endtab %}

{% tab title="Lite" %}
```php
use Enqueue\AmqpExt\AmqpConnectionFactory;

$application = EcotoneLiteApplication::boostrap(
    [
        AmqpConnectionFactory::class => new AmqpConnectionFactory("amqp+lib://guest:guest@localhost:5672//")
    ]
);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
We register our AmqpConnection under the class name `Enqueue\AmqpExt\AmqpConnectionFactory.` This will help Ecotone resolve it automatically, without any additional configuration.
{% endhint %}
