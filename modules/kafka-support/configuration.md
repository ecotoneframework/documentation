# Configuration

## Installation

```php
composer require ecotone/kafka
```

Implementation is based on [rdkafka](https://github.com/arnaud-lb/php-rdkafka).

## Configuration

In order to use **Kafka Support** we need to add **KafkaBrokerConfiguration** to our **Dependency Container.**&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
# config/services.yaml
# You need to have Kafka instance running on your localhost, or change DSN
    Ecotone\Kafka\Configuration\KafkaBrokerConfiguration:
        class: Ecotone\Kafka\Configuration\KafkaBrokerConfiguration
        arguments:
            $bootstrapServers:
                - localhost:9094
```
{% endtab %}

{% tab title="Laravel" %}
```php
# Register Kafka Service in Provider

use Ecotone\Kafka\Configuration\KafkaBrokerConfiguration;

public function register()
{
     $this->app->singleton(KafkaBrokerConfiguration::class, function () {
         return new KafkaBrokerConfiguration(['localhost:9094']);
     });
}
```
{% endtab %}

{% tab title="Lite" %}
```php
use Ecotone\Kafka\Configuration\KafkaBrokerConfiguration;

$application = EcotoneLiteApplication::boostrap(
    [
        KafkaBrokerConfiguration::class => new KafkaBrokerConfiguration(['localhost:9094'])
    ]
);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
We register our **KafkaBrokerConfiguration** under the class name **Ecotone\Kafka\Configuration\KafkaBrokerConfiguration**. This will help Ecotone resolve it automatically, without any additional configuration.
{% endhint %}
