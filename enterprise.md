---
description: Ecotone Enterprise features for scaling multi-tenant and multi-service PHP systems
---

# Enterprise

Ecotone Free gives you production-ready CQRS, Event Sourcing, and Workflows — message buses, aggregates, sagas, async messaging, retries, error handling, and full testing support.

Ecotone Enterprise is for when your system outgrows single-tenant, single-service, or needs advanced resilience and scalability.

## Free vs Enterprise at a Glance

| Capability | Free | Enterprise |
|-----------|------|------------|
| CQRS (Commands, Queries, Events) | Yes | Yes |
| Event Sourcing & Projections | Yes | Yes |
| Sagas (Stateful Workflows) | Yes | Yes |
| Handler Chaining (Pipe & Filter) | Yes | Yes |
| Async Messaging (RabbitMQ, SQS, Redis) | Yes | Yes |
| Retries & Dead Letter | Yes | Yes |
| Outbox Pattern | Yes | Yes |
| Interceptors (Middlewares) | Yes | Yes |
| Testing Support | Yes | Yes |
| Multi-Tenancy | Yes | Yes |
| OpenTelemetry | Yes | Yes |
| Orchestrators | | Yes |
| Distributed Bus with Service Map | | Yes |
| Dynamic Message Channels | | Yes |
| Partitioned Projections | | Yes |
| Blue-Green Deployments | | Yes |
| Kafka Integration | | Yes |
| Command Bus Instant Retries | | Yes |
| Gateway-Level Deduplication | | Yes |

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

### "Our projections need to scale, rebuild safely, or deploy without downtime"

A single global projection can't keep up with event volume. Rebuilding wipes the read model for 30 minutes. Changing projection schema means downtime for users.

* [**Partitioned Projections**](modelling/event-sourcing/setting-up-projections/scaling-and-advanced.md#partitioned-projections) -- One partition per aggregate with independent position tracking. Failures isolate to a single aggregate instead of blocking everything. Indexed event loading skips irrelevant events for dramatically faster processing. Works with both sync and async execution.
* [**Async Backfill & Rebuild**](modelling/event-sourcing/setting-up-projections/backfill-and-rebuild.md) -- Push backfill and rebuild to asynchronous background workers with `asyncChannelName`. Combined with partitioned projections, the work is split into batches that multiple workers process in parallel — throughput scales linearly with worker count. A backfill that takes 2 hours with 1 worker takes 12 minutes with 10.
* [**Blue-Green Deployments**](modelling/event-sourcing/setting-up-projections/blue-green-deployments.md) -- Deploy a new projection version alongside the old one. The new version catches up from history while the old one serves traffic. Switch when ready, delete the old one. Zero downtime.
* [**Streaming Projections**](modelling/event-sourcing/setting-up-projections/scaling-and-advanced.md#streaming-projections) -- Consume events directly from Kafka or RabbitMQ Streams instead of the database event store. For cross-system integration and external event sources.
* [**High-Performance Flush State**](modelling/event-sourcing/setting-up-projections/projections-with-state.md#high-performance-projections-with-flush-state-enterprise) -- Accumulate state in memory across a batch of events and persist once at flush. Process 1000 events with zero database writes, then one bulk insert. Dramatically faster rebuilds.

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

### "We need per-handler control over async endpoint behavior"

Database transactions are globally enabled for your message channel, but some handlers only call a 3rd party API or send emails — wrapping them in a transaction wastes connections and holds locks unnecessarily.

* [**Async Endpoint Annotations**](modelling/asynchronous-handling/asynchronous-message-handlers.md#endpoint-annotations-enterprise) -- Pass `endpointAnnotations` on `#[Asynchronous]` to selectively disable transactions, message collectors, or inject custom configuration for specific handlers while keeping global defaults for the rest of the channel.

### "We need production-grade RabbitMQ consumption"

Custom consumer scripts need manual connection handling, reconnection logic, and shutdown management.

* [**Rabbit Consumer**](modules/amqp-support-rabbitmq/rabbit-consumer.md) -- Set up RabbitMQ consumption with a single attribute. Built-in reconnection, graceful shutdown, and health checks out of the box.

## Materials

### Links

* [Ecotone Enterprise and Kafka, Distributed Bus, Dynamic Channels](https://blog.ecotone.tech/ecotone-enterprise-kafka-distributed-bus-dynamic-channels-and-more-2/) \[Article]
* [Implementing Event-Driven Architecture](https://blog.ecotone.tech/implementing-event-driven-architecture-in-php/) \[Article]
