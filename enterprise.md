# Enterprise

## Ecotone Plans

Ecotone comes with two plans:

* **Ecotone Free** comes with [Apache License Version 2.0](https://github.com/ecotoneframework/ecotone-dev/blob/main/LICENSE) for open features. This covers all the features, which are not marked as Enterprise. \

* **Ecotone Enterprise** is based Enterprise licence. It does provide more advanced features that help building larger scale systems, optimize resource costs and speed up daily development even more than the basic functionality.

{% hint style="success" %}
Each Enterprise feature is marked with hint on the documentation page. Enterprise features can't be run without licence key.\
Enterprise Licence Keys will be sold at **https://ecotone.tech** in early **2025**.\
Stay tuned by [subscribe to mailing list](https://blog.ecotone.tech/#/portal).
{% endhint %}

## Available Enterprise Features

* [**Dynamic Message Channels**](modelling/asynchronous-handling/dynamic-message-channels.md) - Allows for different deployment strategies of Message Channels, provides support for build advanced per Client configurations.
* [**Kafka Support**](modules/kafka-support/) - Enables integration with Kafka (Event Streaming Platform) to send, receive from Messages from topics, and to use Kafka in form of Message Channel abstraction for seamless integration into the System.
* [**Event Sourcing Handlers with Metadata**](modelling/event-sourcing/event-sourcing-introduction/working-with-metadata.md#enterprise-accessing-metadata-during-event-application) -Provides ability to access Event's metadata during Event Sourced Aggregate reconstruction.
* [**Asynchronous Message Buses**](modelling/asynchronous-handling/asynchronous-message-bus-gateways.md) **-** Adds ability to set up Message Bus as asynchronous, which make it easy to create customized solutions like "Outbox Command Bus".
* [**Distributed Bus with Service Map**](modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) - Provides way to communicate between Services (Applications) using all available Message Channels providers (RabbitMQ, Amazon SQS, Redis, Dbal, Kafka, Symfony Message Channels, Laravel Queues).
* Details about more features coming soon...
