---
description: How to build reliable microservice communication in PHP with Ecotone Distributed Bus
---

# Microservice Communication

## The Problem You Recognize

Your monolith is splitting into services. Or you already have multiple services and they need to talk to each other.

The current approach: HTTP calls between services. When Service B is down, Service A fails too. You've built custom retry logic, custom serialization, and custom routing for each service pair. There's no guaranteed delivery — if a request fails, the data is lost unless you built a custom retry mechanism.

The symptoms:
- **Cascading failures** — one service going down takes others with it
- **Custom glue code** per service pair — serialization, routing, error handling
- **No event sharing** — services can't subscribe to each other's events without point-to-point integrations
- **Broker lock-in** — switching from RabbitMQ to SQS means rewriting integration code

## What the Industry Calls It

**Distributed Messaging** — services communicate through a message broker with guaranteed delivery, event sharing, and transport abstraction.

## How Ecotone Solves It

Ecotone's Distributed Bus lets services send commands and publish events to each other through message brokers. Your application code stays the same — Ecotone handles routing, serialization, and delivery:

```php
// Service A: Send a command to Service B
$distributedBus->sendCommand(
    targetServiceName: "order-service",
    command: new PlaceOrder($orderId, $items),
);
```

```php
// Service B: Handle commands from other services — same as local handlers
#[Distributed]
#[CommandHandler]
public function placeOrder(PlaceOrder $command): void
{
    // This handler receives commands from any service
}
```

Supports **RabbitMQ, Amazon SQS, Redis, and Kafka** — swap transports without changing application code.

## Next Steps

- [Distributed Bus](../modelling/microservices-php/distributed-bus/) — Cross-service messaging
- [Message Consumer](../modelling/microservices-php/message-consumer.md) — Consuming from external sources
- [Message Publisher](../modelling/microservices-php/message-publisher.md) — Publishing to external targets

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Distributed Bus with Service Map](../modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) — a topology-aware distributed bus that supports multiple brokers in a single topology, automatic routing, and cross-framework integration.
{% endhint %}
