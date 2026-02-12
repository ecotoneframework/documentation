# Enterprise

## Ecotone Plans

Ecotone comes with two plans:

* **Ecotone Free** comes with [Apache License Version 2.0](https://github.com/ecotoneframework/ecotone-dev/blob/main/LICENSE). It provides everything you need to build message-driven systems in PHP -- CQRS, aggregates, event sourcing, sagas, async messaging, interceptors, and full testing support. This covers all features not marked as Enterprise.
* **Ecotone Enterprise** adds production-grade capabilities for teams whose systems have grown into multi-tenant, multi-service, or high-throughput environments. It brings advanced workflow orchestration, cross-service communication, resilient command handling, and resource optimization.

Every Enterprise licence directly funds continued development of Ecotone's open-source core. When Enterprise succeeds, the entire ecosystem benefits.

{% hint style="success" %}
Each Enterprise feature is marked with hint on the documentation page. Enterprise features can only be run with licence key.\
\
To evaluate Enterprise features, **email us at** "**support@simplycodedsoftware.com**" **to receive trial key**. **Production license keys** are available at [https://ecotone.tech](https://ecotone.tech/pricing).
{% endhint %}

## Signs You're Ready for Enterprise

You don't need Enterprise on day one. These are the growth signals that tell you it's time:

### "We're serving multiple tenants and need isolation"

A noisy tenant's queue backlog shouldn't affect others. Per-tenant scaling shouldn't mean building custom routing infrastructure.

* [**Dynamic Message Channels**](modelling/asynchronous-handling/dynamic-message-channels.md) -- Route messages per-tenant at runtime using header-based or round-robin strategies. Declare the routing once, Ecotone manages the rest. Add tenants by updating the mapping -- no handler code changes.

### "We have complex multi-step business processes"

Business stakeholders ask "what are the steps in this process?" and the answer requires reading multiple files. Adding or reordering steps touches code in many places.

* [**Orchestrators**](modelling/business-workflows/orchestrators.md) -- Define workflow sequences declaratively in one place. Each step is independently testable and reusable. Dynamic step lists adapt to input data without touching step code.

### "We're running multiple services that need to talk to each other"

Building custom inter-service messaging wiring for each service pair has become unsustainable. Different services use different brokers and you need them to communicate.

* [**Distributed Bus with Service Map**](modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) -- Cross-service messaging that supports multiple brokers (RabbitMQ, SQS, Redis, Kafka) in a single topology. Swap transports without changing application code.

### "We need high-throughput event streaming"

RabbitMQ throughput is becoming a bottleneck, or multiple services need to consume the same event stream independently.

* [**Kafka Integration**](modules/kafka-support/) -- Native Kafka support with the same attribute-driven programming model. No separate producer/consumer boilerplate.
* [**RabbitMQ Streaming Channel**](modules/amqp-support-rabbitmq/message-channel.md#rabbitmq-streaming-channel) -- Kafka-like persistent event streaming on existing RabbitMQ infrastructure. Multiple independent consumers with position tracking.

### "Our production system needs to be resilient"

Transient failures cause unnecessary handler failures. Duplicate commands from user retries or webhooks lead to double-processing. Exception handling is scattered across handlers.

* [**Command Bus Instant Retries**](modelling/recovering-tracing-and-monitoring/resiliency/retries.md#customized-instant-retries) -- Recover from transient failures (deadlocks, network blips) with a single `#[InstantRetry]` attribute. No manual retry loops.
* [**Command Bus Error Channel**](modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/#command-bus-error-channel) -- Route failed synchronous commands to dedicated error handling with `#[ErrorChannel]`. Replace scattered try/catch blocks with centralized error routing.
* [**Gateway-Level Deduplication**](modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md#deduplication-with-command-bus) -- Prevent duplicate command processing at the bus level. Every handler behind that bus is automatically protected.

### "We want less infrastructure code in our domain"

Repository injection boilerplate obscures business logic. Every handler follows the same fetch-modify-save pattern. Making the entire bus async requires annotating every handler individually.

* [**Instant Aggregate Fetch**](modelling/command-handling/repository/repository.md#instant-fetch-aggregate) -- Aggregates arrive in your handler automatically via `#[Fetch]`. No repository injection, just business logic.
* [**Event Sourcing Handlers with Metadata**](modelling/event-sourcing/event-sourcing-introduction/working-with-metadata.md#enterprise-accessing-metadata-during-event-application) -- Pass metadata to `#[EventSourcingHandler]` for context-aware aggregate reconstruction without polluting event payloads.
* [**Asynchronous Message Buses**](modelling/asynchronous-handling/asynchronous-message-bus-gateways.md) -- Make an entire command or event bus async with a single configuration change, instead of annotating every handler.

### "We need production-grade RabbitMQ consumption"

Custom consumer scripts need manual connection handling, reconnection logic, and shutdown management.

* [**Rabbit Consumer**](modules/amqp-support-rabbitmq/rabbit-consumer.md) -- Set up RabbitMQ consumption with a single attribute. Built-in reconnection, graceful shutdown, and health checks out of the box.

## Materials

### Links

* [Ecotone Enterprise and Kafka, Distributed Bus, Dynamic Channels](https://blog.ecotone.tech/ecotone-enterprise-kafka-distributed-bus-dynamic-channels-and-more-2/) \[Article]
* [Implementing Event-Driven Architecture](https://blog.ecotone.tech/implementing-event-driven-architecture-in-php/) \[Article]
