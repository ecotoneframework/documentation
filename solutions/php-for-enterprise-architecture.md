---
description: Enterprise architecture patterns in PHP - comparing Ecotone to Spring, Axon, NServiceBus
---

# PHP for Enterprise Architecture

## The Problem You Recognize

You're a technical lead or architect evaluating whether PHP can handle enterprise-grade architecture. Your team knows PHP well, but the business is growing — you need CQRS, Event Sourcing, distributed messaging, multi-tenancy, and production resilience.

The alternative is migrating to Java (Spring + Axon) or .NET (NServiceBus, MassTransit). That means retraining your team, rewriting your application, and losing PHP's development speed.

## PHP Has Grown Up

PHP is no longer just for simple web applications. Modern PHP (8.1+) has union types, enums, fibers, readonly properties, and first-class attributes. Frameworks like Laravel and Symfony provide the web layer. What was missing was the **architecture layer** that lets a system grow from first command handler to event-sourced microservices without rewrites — the equivalent of what Spring Integration and NServiceBus provide in their ecosystems.

Ecotone fills that gap. Built on the same [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) that underpin Spring Integration, NServiceBus, and Apache Camel, Ecotone brings decades-proven architectural patterns to PHP as attribute-driven code on your existing Laravel or Symfony application.

## How Ecotone Compares

| Capability | Java (Axon)           | .NET (NServiceBus) | PHP (Ecotone) |
|-----------|-----------------------|---------------------|--------------|
| CQRS | Yes                   | Yes | Yes |
| Event Sourcing | Yes                   | Manual | Yes |
| Sagas | Yes                   | Yes | Yes |
| Workflow Orchestration | Manual                | Yes | Yes |
| Resiliency (Retries, Dead Letter, Outbox) | Yes                   | Yes | Yes |
| Distributed Messaging | Yes                   | Yes | Yes |
| Multi-Tenancy | Manual                | Manual | Built-in |
| Message Broker Support | Kafka, RabbitMQ, etc. | RabbitMQ, Azure, etc. | RabbitMQ, Kafka, SQS, Redis |
| Observability | Micrometer            | OpenTelemetry | OpenTelemetry |
| Testing Support | Axon Test Fixtures    | NServiceBus Testing | Built-in Test Support |

## What You Get With Ecotone

- **Enterprise Integration Patterns** as the foundation — not a custom abstraction
- **Framework integration** — works on top of Laravel and Symfony, not replacing them
- **Attribute-driven configuration** — PHP 8 attributes instead of XML or YAML
- **Production resilience** — retries, error channels, dead letter, outbox, deduplication
- **Full testing support** — test message flows, aggregates, sagas, and event sourcing in isolation
- **Observability** — OpenTelemetry integration for tracing and metrics
- **Multi-tenancy** — built-in support for tenant-isolated processing

## Next Steps

- [Why Ecotone?](../why-ecotone.md) — Detailed positioning and integration story
- [Installation](../install-php-service-bus.md) — Get started in 5 minutes
- [Enterprise Features](../enterprise.md) — Advanced capabilities for scaling teams
- [Tutorial](../tutorial-php-ddd-cqrs-event-sourcing/) — Hands-on learning path
