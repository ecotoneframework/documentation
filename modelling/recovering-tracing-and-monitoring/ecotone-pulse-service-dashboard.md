# Ecotone Pulse (Service Dashboard)

<figure><img src="../../.gitbook/assets/Screenshot from 2022-09-23 21-11-16 (2).png" alt=""><figcaption><p>View your Error Messages and Replay them when fix directly from the Dashboard</p></figcaption></figure>

Whenever message fails during [asynchronous processing](../asynchronous-handling/) it will kept repeated till the moment it will succeed. However retry strategy with dead letter queue may be set up in order to retry message given amount of times and then move it to the storage for later review and manual retry.

This is where Ecotone Pulse kicks in, as instead of reviewing and replaying the message directly from the application's console, you may do it directly from the UI application. Besides you may connect multiple Ecotone's application to the Pulse Dashboard to have full overview of your whole system.

{% hint style="success" %}
Ecotone Pulse provide way to control error messages for all your services from one place.
{% endhint %}

## Installation

Enable [Dead Letter](resiliency/error-channel-and-dead-letter/) in your service and [Distributed Consumer](../../modules/amqp-support-rabbitmq.md#distributed-consumer) with [AMQP Module](../../modules/amqp-support-rabbitmq.md#installation).

After this you may run docker image with Ecotone Pulse passing the configuration to your services and RabbitMQ connection.

Then run docker image with Ecotone Pulse passing environment variables:

```
docker run -p 80:80 -e SERVICES='[{"name":"customer_service","databaseDsn":"mysql://user:pass@host/db_name"}]' -e AMQP_DSN='amqp://guest:guest@rabbitmq:5672//' -e APP_DEBUG=true ecotoneframework/ecotone-pulse:0.1.0
```

### SERVICES environment

```
SERVICES=[{"name":"customer_service","databaseDsn":"mysql://user:pass@host/db_name"}]
```

Provide array of services with service name and [database connection dsn](../../modules/dbal-support.md#installation).

{% hint style="info" %}
The `name` in ServiceName from your Symfony/Laravel/Lite configuration.\
This way Ecotone Pulse knows how to route messages to your Service.
{% endhint %}

### AMQP\_DSN environment

```
AMQP_DSN='amqp://guest:guest@rabbitmq:5672//'
```

DSN to your RabbitMQ instance, which services are connected with [Distributed Consumer](../../modules/amqp-support-rabbitmq.md#distributed-consumer).

{% hint style="info" %}
It's important to set up Amqp Distributed Consumer. This way Service starts to subscribe to messages coming from Ecotone Pulse.
{% endhint %}

## Usage

![](<../../.gitbook/assets/Screenshot from 2022-09-23 21-11-16.png>)

In the dashboard you may check all the connected services. For quick overview, you will find amount of errors within given service there.

![](<../../.gitbook/assets/Screenshot from 2022-09-23 21-11-22.png>)

To review error messages go to specific service. From there you can review the error message, stacktrace and replay it or delete.

## Demo Application

You may check demo application, where Symfony and Laravel services are connected to Ecotone pulse in [demo application](https://github.com/ecotoneframework/php-ddd-cqrs-event-sourcing-symfony-laravel-ecotone).
