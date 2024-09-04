# Business First

<figure><img src=".gitbook/assets/ecotone_logo_no_background (2).png" alt="" width="563"><figcaption></figcaption></figure>

## About

**Ecotone** promotes way of developing PHP Applications that have it's roots in **Message-Driven Architecture** combined with **Domain Driven Design**.\
It helps deliver business focused software quickly, by providing foundation which guarantee decoupling, reliability and scalability.

Ecotone brings powerful concepts like **Messaging** based on [**Enterprise Integration Patterns**](https://www.enterpriseintegrationpatterns.com/) **with** **CQRS and Message Bus support,** and well known DDD building blocks like **Aggregates, Sagas and Event Sourcing**.

{% hint style="success" %}
Ecotone just as Frameworks from other programming languages ([NServiceBus](https://particular.net/nservicebus), [Axon Framework](https://docs.axoniq.io/reference-guide/), and [Spring Integration](https://spring.io/projects/spring-integration)) introduces implementation of [**Enterprise Integration Patterns**](https://www.enterpriseintegrationpatterns.com/)**.** \
This allows us to build systems in a way, which connects different components, modules or even Applications in a decoupled, seamless way. Together with higher level DDD patterns, it creates environment in which building complex workflows and business logic becomes really easy to implement and maintain in the long term.
{% endhint %}

## Business Oriented Architecture

[Ecotone](https://blog.ecotone.tech/revolutionary-boa-framework-ecotone/) embrace the concept of Business-Oriented Architecture, which follows fundamental principle of making business logic the primary citizen in our Applications. It shifts the focus from technical details to the actual business processes. \
**Business Oriented Architecture** is built around three main pillars:

<figure><img src=".gitbook/assets/image (4).png" alt="" width="563"><figcaption><p>When all thee pillars are solved by Ecotone, what is left to write is Business Oriented Code</p></figcaption></figure>

1. **Resilient Messaging** **-** At the heart of Ecotone lies a resilient messaging system that enables loose coupling, fault tolerance, and self-healing capabilities.
2. **Declarative Configuration -** Introduces declarative programming which simplifies development, reduces boilerplate code, and promotes code readability. It empowers developers to express their intent clearly, resulting in more maintainable and expressive codebases.
3. **Building Blocks -** Building blocks like Message Handlers, Aggregates, Sagas, facilitate the implementation of the business logic. By making it possible to bind Building Blocks with Resilient Messaging, Ecotone makes it easy to build and connect even the most complex business workflows.

By providing all those three Pillars, Ecotone provides an foundation which then allows us to fully focus on the business side of the things.

{% hint style="success" %}
To start with Ecotone there is no need for big impact refactor. You may introduce it in your existing code base and start using it from day one, even for a single feature.&#x20;

Ecotone works out of the box with popular PHP frameworks like [Symfony](modules/symfony/symfony-ddd-cqrs-event-sourcing.md), [Laravel](modules/laravel/laravel-ddd-cqrs-event-sourcing.md) and can be run stand alone or with any other framework (e.g. Laminas, CodeIgniter, Magento) using [Ecotone Lite](modules/ecotone-lite/).
{% endhint %}

## How to get started

Check how to [install](install-php-service-bus.md) **Ecotone** for **Symfony**, **Laravel** or **Lite.**

* [Learn by example](quick-start-php-ddd-cqrs-event-sourcing/)
* [Go through tutorial](tutorial-php-ddd-cqrs-event-sourcing/)
* [Workshops, Support, Consultancy](other/contact-workshops-and-support.md)
