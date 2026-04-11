---
description: Implementing Microservices and Event Driven Architecture in PHP
---

# Distributed Bus and Microservices

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

You're calling other services via HTTP. When Service B is down, Service A fails too. You've built custom retry logic, custom serialization, and custom routing for each service pair. There's no event sharing — services can't subscribe to each other's events without point-to-point integrations.

## How Ecotone Solves It

Ecotone's **Distributed Bus** provides cross-service messaging through message brokers (RabbitMQ, SQS, Redis, Kafka). Services send commands and publish events to each other with guaranteed delivery. Swap transports without changing application code.

---

Ecotone comes with Support for integrating Service (Applications) together in decoupled way, for this Ecotone provides `Distributed Bus`.

Read more in bellow sections:

{% content-ref url="distributed-bus/" %}
[distributed-bus](distributed-bus/)
{% endcontent-ref %}

{% content-ref url="message-consumer.md" %}
[message-consumer.md](message-consumer.md)
{% endcontent-ref %}

{% content-ref url="message-publisher.md" %}
[message-publisher.md](message-publisher.md)
{% endcontent-ref %}

## Materials

### Demo implementation

* Simple demo [using Ecotone Lite](https://github.com/ecotoneframework/quickstart-examples/tree/main/Microservices).
* Advanced demo [using Ecotone Lite](https://github.com/ecotoneframework/quickstart-examples/tree/main/MicroservicesAdvanced).
* Symfony and Laravel [application integration](https://github.com/ecotoneframework/php-ddd-cqrs-event-sourcing-symfony-laravel-ecotone).

### Links

* [Implementing Event-Driven Architecture](https://blog.ecotone.tech/implementing-event-driven-architecture-in-php/) \[Article]
* [Ecotone Enterprise and Distributed Bus with Service Mapping](https://blog.ecotone.tech/ecotone-enterprise-kafka-distributed-bus-dynamic-channels-and-more-2/) \[Article]
* [Starting with Microservices in PHP](https://blog.ecotone.tech/how-to-integrate-microservices-in-php/) \[Article]
* [Loosely coupled Microservices in PHP](https://blog.ecotone.tech/loosely-coupled-microservices-in-php/) \[Article]
