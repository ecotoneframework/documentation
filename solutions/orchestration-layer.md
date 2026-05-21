---
description: The PHP orchestration layer for distributed systems — Orchestrators, Sagas, Distributed Bus, Service Map, and EIP primitives on Laravel and Symfony
---

# The PHP Orchestration Layer

For a PHP team that needs to *orchestrate* — multi-step business processes, cross-service messaging between PHP applications, EIP-style routing of messages by content or context, multi-tenant traffic shaping in one deployment — Ecotone delivers declarative orchestration through PHP attributes, on the database and broker you already operate, integrated with Laravel and Symfony.

This page covers the orchestration story from three angles: process orchestration (Orchestrators, Sagas, chained workflows), service-to-service orchestration (Distributed Bus, Service Map), and message-routing orchestration (EIP primitives). The comparison closes with how Ecotone differs from BPMN engines like Camunda/Zeebe and the "stitch it from Messenger + state machines" alternative.

## The Problem You Recognize

Three different versions of *"where is the logic?"* — the same architectural failure mode in different costumes.

**The process problem.** Order fulfillment spans payment + inventory + shipping + notification, with branching (digital vs physical fulfillment), waiting (24-hour timeouts), and compensation (refund + restock on cancellation). The orchestration is spread across event listeners that trigger event listeners, cron jobs scanning `is_processed` columns, and a `step_completed_at` table. Nobody can explain the full process without reading every file.

**The service problem.** Three PHP services need to communicate. You've built HTTP calls between them, custom serialization per pair, custom retry logic, and the question *"how do new subscribers learn about events from Service A?"* has no answer that doesn't involve another point-to-point integration.

**The routing problem.** Messages need to fan out: one event triggers four handlers, each on a different broker, each with different priority and retry policy, some tenant-routed, some not. Symfony Messenger or Laravel Queues plus custom middleware can do this — eventually — but the routing logic ends up spread across YAML config, middleware classes, and service-tag wiring.

## What the Industry Calls It

**Orchestration Layer** — the architectural tier that sits between application code and message infrastructure, declaring *how* messages flow, *which* steps compose into processes, and *where* messages cross service boundaries. In Java this layer is Spring Integration or Axon; in .NET it's NServiceBus, MassTransit, or Wolverine. In PHP the equivalent vocabulary — Enterprise Integration Patterns expressed as PHP 8 attributes — is what Ecotone provides.

## How Ecotone Solves It

### Orchestrators — declarative multi-step workflows (Enterprise)

The `#[Orchestrator]` attribute defines a workflow as a sequence of channel names. Each step is a `#[InternalHandler]` independently testable and reusable. The orchestrator method returns the channel list — including dynamic step lists chosen from input data.

```php
final class OrderFulfillmentOrchestrator
{
    #[Orchestrator(inputChannelName: 'order.fulfill')]
    public function plan(PlaceOrder $order): array
    {
        return $order->isDigital()
            ? ['order.charge', 'order.deliver_digital', 'order.notify']
            : ['order.charge', 'order.reserve_stock', 'order.ship', 'order.notify'];
    }

    #[InternalHandler(inputChannelName: 'order.charge')]
    public function charge(PlaceOrder $order, PaymentService $payments): PlaceOrder { /* ... */ return $order; }

    #[InternalHandler(inputChannelName: 'order.ship')]
    public function ship(PlaceOrder $order, ShippingService $shipping): PlaceOrder { /* ... */ return $order; }

    // ...
}
```

The orchestration logic lives in one method that a business stakeholder can read. Each step is a plain handler.

### Sagas — stateful process managers

When the process spans events arriving over time (payment received → ship → notify, with hours or days between steps), use a `#[Saga]`. State is persisted per `#[Identifier]`; on the next event arrival, the saga reloads and reacts. `#[Delayed]` handlers fire timeouts without cron.

### Stateless workflows — chained handlers

When the message *is* the state, chain handlers via `outputChannelName`. No saga record needed; durability comes from the channel (outbox + redelivery).

### Distributed Bus + Service Map — service-to-service orchestration

```php
$distributedBus->sendCommand(
    targetServiceName: 'billing-service',
    command: new IssueRefund($orderId, $amount),
);
```

```php
#[Distributed]
#[CommandHandler]
public function issueRefund(IssueRefund $command): void { /* runs on the billing service */ }
```

The Distributed Bus moves commands and events between PHP services over the brokers you already operate (RabbitMQ, Kafka, SQS, Redis). The Service Map (Enterprise) carries the topology — which service consumes which routing keys, on which broker — so adding a service is a config change, not a code change in every caller. Multi-broker single-topology: some services on Kafka, others on RabbitMQ, all coordinated through the Service Map.

### EIP primitives — composable message routing

Routers (route by payload or header), splitters (one message → many), filters, enrichers, transformers — all as PHP attributes on plain methods. The Enterprise Integration Patterns vocabulary, expressed as code, not XML or YAML.

```php
#[Router('route.payment')]
public function route(PlaceOrder $order): string
{
    return $order->amount > 10_000 ? 'orders.high_value' : 'orders.standard';
}

#[Splitter(inputChannelName: 'order.fulfill', outputChannelName: 'item.ship')]
public function splitItems(PlaceOrder $order): array
{
    return $order->items;
}
```

### Multi-tenant routing in one deployment

Header-routed channels resolve at runtime: a `tenant` header on a message routes it to the tenant-specific channel. One deployment, N tenants, isolated processing — without one namespace per tenant or one cluster per tenant.

## How It Compares

| Dimension | Camunda / Zeebe + PHP client | Symfony Messenger + custom state machine | Ecotone |
| --- | --- | --- | --- |
| Runtime | Java cluster you operate | PHP process | PHP process — the database and broker you already operate |
| Workflow definition | BPMN diagram + external PHP client integration | Hand-written state machine, often a Laravel package + custom YAML | Declarative `#[Orchestrator]` in PHP, one method, one file |
| Cross-service communication | Camunda Worker pattern (PHP polls Camunda) | HTTP calls or Messenger transports per pair, hand-rolled retry/serialization | Distributed Bus + Service Map, multi-broker single topology |
| EIP primitives (routers, splitters, filters) | Not the right shape | Hand-rolled middleware | First-class attributes |
| Multi-tenant routing | Manual per-tenant deployments / namespaces | Custom middleware per project | Header-routed channels, dynamic resolution |
| Resiliency primitives shared across orchestrator + handlers | Two separate models | Each library/package its own retry story | One model — retry, error channel, DLQ, dedup, OTel uniform across orchestrator steps and standalone handlers |
| Integration with Laravel / Symfony | External system | Symfony-native; Laravel needs glue | Bundle for Symfony, Provider for Laravel, Lite for any PSR-11 container |

Camunda/Zeebe are mature BPMN engines; if your organisation already operates a JVM cluster and stakeholders want BPMN diagrams, they remain the right answer at the engine level. One PHP-specific factual point: as of 2026, no actively-maintained Camunda PHP client exists — the most-discoverable community SDK (`tistre/camunda_php_client`) was archived in February 2025. For PHP teams that don't want to add a Java cluster, that don't have an actively-maintained PHP-side SDK, and for the "stitch it from primitives" path that ends in six libraries with no shared retry or PII story, Ecotone covers the layer.

## Next Steps

- [Orchestrators](../modelling/business-workflows/orchestrators.md) — declarative multi-step workflows (Enterprise)
- [Sagas](../modelling/business-workflows/sagas.md) — stateful process managers
- [Connecting Handlers with Channels](../modelling/business-workflows/connecting-handlers-with-channels.md) — stateless workflows
- [Distributed Bus](../modelling/microservices-php/distributed-bus/) — cross-service messaging
- [Distributed Bus with Service Map](../modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) — topology-aware distribution (Enterprise)
- [Multi-tenancy Support](../messaging/multi-tenancy-support/) — header-routed channels
- [EIP Routing](../modelling/extending-messaging-middlewares/) — routers, splitters, filters
- [Complex Business Processes](complex-business-processes.md) — workflow + saga side by side
- [Microservice Communication](microservice-communication.md) — service-to-service patterns

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Orchestrators](../modelling/business-workflows/orchestrators.md) with dynamic step lists, [Distributed Bus with Service Map](../modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) for multi-broker topology, [Dynamic Message Channels](../messaging/multi-tenancy-support/) for tenant-routed traffic, and [Streaming Projections](../modelling/event-sourcing/setting-up-projections/scaling-and-advanced.md) over Kafka / RabbitMQ Streams.
{% endhint %}
