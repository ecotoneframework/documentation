# Enterprise

## Ecotone Plans

Ecotone comes with two plans:

* **Ecotone Free** comes with [Apache License Version 2.0](https://github.com/ecotoneframework/ecotone-dev/blob/main/LICENSE) for open features. It allows to build message-driven system in PHP, which solves resiliency and scalability at the architecture level. This covers all the features, which are not marked as Enterprise. \

* **Ecotone Enterprise** is based Enterprise licence. It does provides more advanced set of features aim for Enterprise usage. It does bring to the table custom features, additional integrations, and ability to optimization resource usages.&#x20;

{% hint style="success" %}
Each Enterprise feature is marked with hint on the documentation page. Enterprise features can only be run with licence key.\
\
Subscription plan is now available. To subscribe to Enterprise visit [https://ecotone.tech/pricing](https://ecotone.tech/pricing)
{% endhint %}

## Available Enterprise Features

* [**Dynamic Message Channels**](modelling/asynchronous-handling/dynamic-message-channels.md) - Provides ability to simplify deployment strategy, adjusting asynchronous processing to business scenarios, and configure processing per Client dynamically (which is especially useful in Multi-Tenant and SAAS environments).
* [**Kafka Support**](modules/kafka-support/) - Enables integration with Kafka (Event Streaming Platform) to send, receive from Messages from topics, and to use Kafka in form of Message Channel abstraction for seamless integration into the System.
* [**Event Sourcing Handlers with Metadata**](modelling/event-sourcing/event-sourcing-introduction/working-with-metadata.md#enterprise-accessing-metadata-during-event-application) - Provides ability to pass Metadata to Aggregate's Event Sourcing Handlers. This can be used to to adjust Aggregate's reconstruction process, based on Metadata information stored in related Events.
* [**Asynchronous Message Buses**](modelling/asynchronous-handling/asynchronous-message-bus-gateways.md) **-** This grants ability to build customized Command/Event Buses where Message will first go over given Asynchronous Channel. This can be used to build for example Outbox Command Bus.
* [**Distributed Bus with Service Map**](modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) - Provides way to communicate between Services (Applications) with ease and in explicit and decoupled way. Make it possible to use all available Message Channels providers (RabbitMQ, Amazon SQS, Redis, Dbal, Kafka, Symfony Message Channels, Laravel Queues).
* Details about more features coming soon...

## Materials

### Links

* [Ecotone Enterprise and Kafka, Distributed Bus, Dynamic Channels](https://blog.ecotone.tech/ecotone-enterprise-kafka-distributed-bus-dynamic-channels-and-more-2/) \[Article]
