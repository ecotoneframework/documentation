# Configuration

## Service Name Configuration

In order for `Ecotone` how to route messages you need to register Service Name (Application Name).

* [Symfony Service Name Configuration](../../../../modules/symfony/symfony-ddd-cqrs-event-sourcing.md#servicename)
* [Laravel Service Name Configuration](../../../../modules/laravel/laravel-ddd-cqrs-event-sourcing.md#servicename)
* [Ecotone Lite Service Name Configuration](../../../../modules/ecotone-lite/#servicename)&#x20;

## Configure Distribution

To create [AMQP Distributed configuration](../../) use [Service Context](../../../../messaging/service-application-configuration.md).

### Distributed Message Publisher

This will register **DistributedBus** in Dependency Container, which can be used to send Distributed Messages:

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function distributedPublisher()
    {
        return AmqpDistributedBusConfiguration::createPublisher();
    }
}
```

After that **DistributedBus** will become available in Dependency Container, ready to start sending Messages.

### Distributed Message Consumer

This will enable Message Consumer with Service Name, which can consume Distributed Messages:

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function distributedConsumer()
    {
        return AmqpDistributedBusConfiguration::createConsumer();
    }
}
```

### Run the consumer

Run consumer for your registered distributed consumer. It will be available under your [Service Name](../../#configuration)

List:

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:list
+--------------------+
| Endpoint Names     |
+--------------------+
| billing            |
+--------------------+
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:list
+--------------------+
| Endpoint Names     |
+--------------------+
| billing            |
+--------------------+
```
{% endtab %}

{% tab title="Lite" %}
```
$consumers = $messagingSystem->list()
```
{% endtab %}
{% endtabs %}

Run it:

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:run billing -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```
artisan ecotone:run billing -vvv
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->run("billing");
```
{% endtab %}
{% endtabs %}
