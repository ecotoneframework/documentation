---
description: Why Ecotone - The enterprise architecture layer for PHP
---

# Why Ecotone?

## What Ecotone Is (and Isn't)

Ecotone is **not** a framework replacement. You don't rewrite your Laravel or Symfony application to use Ecotone — you add it.

Think of it this way: API Platform provides the API layer on top of Symfony. **Ecotone provides the enterprise messaging layer on top of your framework.**

You keep your ORM (Eloquent or Doctrine), your routing, your templates, your deployment. Ecotone handles the messaging architecture — the part that makes your application resilient, scalable, and maintainable as complexity grows.

```bash
# That's it. Your framework stays, Ecotone adds the enterprise layer.
composer require ecotone/laravel
# or
composer require ecotone/symfony-bundle
```

## Every Ecosystem Has This Layer — Except PHP

Enterprise software in other ecosystems has mature tooling for messaging, CQRS, Event Sourcing, and distributed systems:

| Ecosystem | Enterprise Messaging Layer |
|-----------|--------------------------|
| **Java** | Spring + Axon Framework |
| **.NET** | NServiceBus, MassTransit, Wolverine |
| **PHP** | **Ecotone** |

This isn't about PHP being inferior. It's about PHP maturing into enterprise domains. Teams building complex business systems in PHP deserve the same caliber of tooling that Java and .NET teams have had for years.

Ecotone is built on the same foundation — [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) — that powers Spring Integration, NServiceBus, and Apache Camel.

## What You Get

Instead of learning pattern names first, start with the problem you're solving:

| Your problem | What the industry calls it | Ecotone feature |
|-------------|---------------------------|-----------------|
| Business logic is scattered across controllers and services | CQRS (Command Query Responsibility Segregation) | [Message Bus and CQRS](modelling/command-handling/) |
| You need a full audit trail and the ability to rebuild state | Event Sourcing | [Event Sourcing](modelling/event-sourcing/) |
| Complex multi-step processes are hard to follow and maintain | Sagas, Workflow Orchestration | [Business Workflows](modelling/business-workflows/) |
| Async processing is unreliable and hard to debug | Resilient Messaging | [Resiliency](modelling/recovering-tracing-and-monitoring/resiliency/) |
| Services need to communicate reliably across boundaries | Distributed Messaging | [Distributed Bus](modelling/microservices-php/) |
| Multiple tenants need isolated processing | Multi-Tenancy | [Multi-Tenancy Support](messaging/multi-tenancy-support/) |

## How It Integrates

Ecotone plugs into your existing framework without requiring changes to your application structure:

### Laravel

Laravel's queue runs jobs, not business processes — anything resembling aggregates, sagas, workflows, or event sourcing ends up stitched together from separate libraries. Ecotone fills that layer directly: works with **Eloquent** for aggregate persistence, **Laravel Queues** for async message channels, and **Laravel Octane** for high-performance scenarios. Configuration via your standard Laravel config files.

```bash
composer require ecotone/laravel
```

[Laravel Module Documentation](modules/laravel/)

### Symfony

Symfony Messenger handles dispatch — aggregates, sagas, event sourcing, and transactional outbox are left to you. Ecotone fills that layer directly: works with **Doctrine ORM** for aggregate persistence, **Symfony Messenger Transport** for async message channels, and standard **Bundle configuration**. Ecotone auto-discovers your attributes in the `src` directory.

```bash
composer require ecotone/symfony-bundle
```

[Symfony Module Documentation](modules/symfony/)

### Standalone

For applications without Laravel or Symfony, Ecotone Lite provides the full feature set with minimal dependencies.

```bash
composer require ecotone/lite-application
```

[Ecotone Lite Documentation](modules/ecotone-lite/)

## Start Free, Scale with Enterprise

**Ecotone Free** gives you everything you need for production-ready CQRS, Event Sourcing, and Workflows — message buses, aggregates, sagas, async messaging, interceptors, retries, error handling, and full testing support.

**Ecotone Enterprise** is for when your system outgrows single-tenant, single-service, or needs advanced resilience and scalability — orchestrators, distributed bus with service map, dynamic channels, partitioned projections, Kafka integration, and more.

[Learn about Enterprise features](enterprise.md)
