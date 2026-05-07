---
description: Deriving the tenant header from external messages that do not carry tenant metadata
---

# Deriving Tenant from Inbound Messages

{% hint style="info" %}
**Enterprise feature.** `#[WithTenantResolver]` requires an Ecotone Enterprise licence.
{% endhint %}

## The Problem

When messages enter our application from outside — a Kafka topic, an AMQP queue, a scheduled poller pulling from a third-party API — they often **don't carry the tenant header** that the rest of the system relies on for connection routing. The information is there (the originating topic, a payload field, a queue name), it just hasn't been translated into Ecotone's tenant header yet.

Trying to handle this with a normal `#[Before]` interceptor or `#[AddHeader]` is brittle: by the time those run, the connection-switching interceptors have already fired and thrown _"Lack of context about tenant in Message Headers"._

## The Solution

Place `#[WithTenantResolver(expression: ...)]` on the **inbound channel adapter method** — the method that consumes externally-arriving messages. The expression is evaluated against the inbound message _before_ any tenant-aware interceptor sees it, and its result is written into the tenant header.

```php
final class OrderEventConsumer
{
    #[KafkaConsumer('orders', topics: ['orders_eu', 'orders_us'])]
    #[WithTenantResolver(expression: "headers['kafka_topic']")]
    public function handle(string $payload, #[Headers] array $headers): void
    {
        // headers['tenant'] is now populated from the originating Kafka topic
    }
}
```

The expression has access to `payload` and `headers` of the inbound message. Whatever it returns is injected as the tenant header (whose name comes from your `MultiTenantConfiguration`). Downstream tenant-aware interceptors then see the correct tenant and route the connection accordingly — your handler code remains unchanged.

## Where It Can Be Placed

`#[WithTenantResolver]` is only valid on **inbound channel adapter methods**, where messages may arrive from outside the application without a tenant header:

* `#[KafkaConsumer]`
* `#[AmqpConsumer]`
* `#[Scheduled]`
* Any other inbound channel adapter

Placing it on a synchronous or asynchronous `#[CommandHandler]`, `#[EventHandler]`, or `#[QueryHandler]` is rejected at boot with a `ConfigurationException`. Internal Message Channels — including those used by async handlers — already carry the tenant context propagated from the originating bus call, so a resolver there has nothing to derive from.

If your async handler is processing externally-arrived messages, attach `#[WithTenantResolver]` to the inbound adapter that produces them, not to the handler itself.

## Scheduled Pollers

The same attribute works for `#[Scheduled]` methods that pull events from external sources:

```php
final class ExternalEventPoller
{
    public function __construct(private readonly PendingEventSource $source) {}

    #[Scheduled(requestChannelName: 'externalEventArrived', endpointId: 'externalPoller')]
    #[WithTenantResolver(expression: "payload['tenantId']")]
    public function poll(): ?Message
    {
        $event = $this->source->next();
        return $event === null
            ? null
            : MessageBuilder::withPayload($event['data'])
                ->setHeader('tenantId', $event['tenantId'])
                ->build();
    }
}
```

## Looking Up Tenant Through External Mapping

The expression can reference any service registered in the container. This is useful when the originating identifier (a topic, a queue, a code) does not literally equal the tenant name and needs to be translated through a mapping service:

```php
#[KafkaConsumer('orders', topics: ['orders_topic_1', 'orders_topic_2'])]
#[WithTenantResolver(expression: "reference('topicTenantMap').lookup(headers['kafka_topic'])")]
public function handle(string $payload, #[Headers] array $headers): void
{
    // ...
}
```

`topicTenantMap` is just a regular service in your container. You can hold the mapping in configuration, in a database table, or anywhere else.

## Explicit Tenant Header Wins

If the inbound message _already_ carries the tenant header (for example, an upstream Kafka producer sets it explicitly), the resolver does **not** override it. Explicit headers always take precedence. This makes `#[WithTenantResolver]` safe to add even when only _some_ of your incoming messages need translation.

## Null Result

If the expression evaluates to `null`, no tenant header is added — and downstream tenant-aware code will fail loudly with the standard _"Lack of context about tenant"_ exception. The resolver does not invent a default; misconfiguration surfaces as a clear failure rather than silent routing to an arbitrary tenant.

## Non-Scalar Result

If the expression evaluates to something other than `string`, `int`, or `null` (for example, an object or array), the resolver throws an `InvalidArgumentException` naming the expression and the actual type. Tenant identifiers are always scalars; this catches misconfigured expressions early.

{% hint style="success" %}
`#[WithTenantResolver]` lets you keep multi-tenancy concerns at the system boundary — exactly where external messages enter your application. Downstream handler code stays the same as in non-multi-tenant systems, just like the rest of Ecotone's multi-tenancy story.
{% endhint %}
