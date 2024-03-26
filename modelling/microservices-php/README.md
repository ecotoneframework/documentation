---
description: Implementing Microservices in PHP
---

# Distributed Bus and MicroServices

Ecotone comes with Support for integrating Service together in decoupled way.\
For this Ecotone provides `Distributed Bus` in case all services work in PHP.\
Or in case of integrating non PHP or when `Distributed Bus` can't be used, then `Message Consumer` and `Message Publisher` can be used instead.

{% hint style="success" %}
You may consider integrating service using Message Channels, however Message Channels are providing both sides consuming and publishing. \
\
Consider `Message Channels` for communication on the level of the same Service. \
`Message Publishers` and `Message Consumers` to integrate with other Services.&#x20;
{% endhint %}

{% content-ref url="distributed-bus.md" %}
[distributed-bus.md](distributed-bus.md)
{% endcontent-ref %}

{% content-ref url="message-consumer.md" %}
[message-consumer.md](message-consumer.md)
{% endcontent-ref %}

{% content-ref url="message-publisher.md" %}
[message-publisher.md](message-publisher.md)
{% endcontent-ref %}

## Materials

### Demo implementation

* Simple demo [using Ecotone Lite](https://github.com/ecotoneframework/quickstart-examples/tree/main/Microservices).&#x20;
* Advanced demo [using Ecotone Lite](https://github.com/ecotoneframework/quickstart-examples/tree/main/MicroservicesAdvanced).
* Symfony and Laravel [application integration](https://github.com/ecotoneframework/php-ddd-cqrs-event-sourcing-symfony-laravel-ecotone).

### Links

* [Integrating PHP Applications using Distributed Bus with RabbitMQ](https://blog.ecotone.tech/integrating-php-applications-with-ecotone-and-rabbitmq/) \[Article]
* [Starting with Microservices in PHP](https://blog.ecotone.tech/how-to-integrate-microservices-in-php/) \[Article]
* [Loosely coupled Microservices in PHP](https://blog.ecotone.tech/loosely-coupled-microservices-in-php/) \[Article]
