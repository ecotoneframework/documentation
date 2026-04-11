# Enterprise Repositioning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reposition the Ecotone documentation site from pattern-first to problem-first, framing Ecotone as the enterprise architecture layer for Laravel and Symfony.

**Architecture:** Content-only changes to a GitBook documentation site. All files are markdown. New pages follow existing GitBook conventions (frontmatter with `description` field, GitBook hint blocks via `{% hint %}` syntax). No code compilation or tests — verification is visual review of rendered markdown.

**Tech Stack:** GitBook, Markdown, Git

---

### Task 1: Home Page Overhaul (README.md)

**Files:**
- Modify: `README.md` (full rewrite)

- [ ] **Step 1: Rewrite README.md with new positioning**

Replace the entire content of `README.md` with:

```markdown
# About

<figure><img src=".gitbook/assets/ecotone_logo_no_background (1).png" alt="" width="563"><figcaption></figcaption></figure>

## CQRS, Event Sourcing, Workflows, and Production Resilience for Laravel and Symfony

Add production-grade architecture to your existing framework with a single Composer package. No framework change. No base classes. Just PHP attributes.

---

## What problem does Ecotone solve?

### Your app grew, your architecture didn't

Controllers doing too much. Business logic scattered across listeners, services, and middleware. Every change risks breaking something unrelated. Testing requires bootstrapping the entire framework.

**Ecotone introduces clear separation** — Command Handlers for writes, Query Handlers for reads, Event Handlers for reactions. Each has a single responsibility, wired automatically through PHP attributes.

### Async processing is fragile

Failed jobs disappear silently. There's no retry strategy beyond "try again 3 times." You can't replay failed messages or see what's stuck in the queue. Going async required touching every handler.

**Ecotone provides production resilience** — automatic retries, error channels, dead letter queues, outbox pattern, and idempotency. All declarative via attributes, with a single attribute to make any handler async.

### Enterprise patterns feel out of reach in PHP

You've seen CQRS, Event Sourcing, and Sagas in Java and .NET ecosystems. Every PHP implementation requires rewriting your application or adopting an opinionated framework.

**Ecotone brings these patterns to your existing stack** — works with Laravel (Eloquent, Queues, Octane) and Symfony (Doctrine, Messenger Transport) without requiring a framework change. No base classes to extend, no interfaces to implement.

---

## Choose your path

**Laravel developers** — Keep Eloquent, add enterprise messaging.\
Start with [Laravel Quick Start](quick-start-php-ddd-cqrs-event-sourcing/laravel-ddd-cqrs-demo-application.md) or [Laravel Module docs](modules/laravel/).

**Symfony developers** — Go beyond Symfony Messenger.\
Start with [Symfony Quick Start](quick-start-php-ddd-cqrs-event-sourcing/symfony-ddd-cqrs-demo-application/) or [Symfony Module docs](modules/symfony/).

**Architects** — PHP can do what Spring and Axon do.\
Read [Why Ecotone?](why-ecotone.md) to understand the positioning.

---

## How it works

1. **Install via Composer** — `composer require ecotone/laravel` or `ecotone/symfony-bundle`
2. **Add attributes to your code** — Mark methods as Command Handlers, Event Handlers, or Queries using PHP attributes
3. **Ecotone wires the messaging** — Message buses, async channels, retries, and event sourcing are handled automatically

---

## Get started in minutes

* [Install](install-php-service-bus.md) for Symfony, Laravel, or any PHP framework
* [Learn by example](quick-start-php-ddd-cqrs-event-sourcing/) — Send your first command in 5 minutes
* [Go through tutorial](tutorial-php-ddd-cqrs-event-sourcing/) — Build a complete messaging flow step by step
* [Workshops, Support, Consultancy](other/contact-workshops-and-support.md) — Hands-on training for your team

{% hint style="info" %}
Built on [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/). Used in production by teams running multi-tenant, event-sourced systems at scale.
{% endhint %}

{% hint style="success" %}
Join [Ecotone's Community Channel](https://discord.gg/GwM2BSuXeg), and ask questions there.
{% endhint %}
```

- [ ] **Step 2: Review the rendered markdown**

Open `README.md` and verify:
- The logo image path still works (`.gitbook/assets/ecotone_logo_no_background (1).png`)
- All internal links resolve to existing files
- GitBook hint blocks use correct syntax

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "Rewrite home page with enterprise positioning and problem-first framing"
```

---

### Task 2: New "Why Ecotone?" Page

**Files:**
- Create: `why-ecotone.md`

- [ ] **Step 1: Create the Why Ecotone page**

Create `why-ecotone.md` in the repository root:

```markdown
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

Works with **Eloquent** for aggregate persistence, **Laravel Queues** for async message channels, and **Laravel Octane** for high-performance scenarios. Configuration via your standard Laravel config files.

```bash
composer require ecotone/laravel
```

[Laravel Module Documentation](modules/laravel/)

### Symfony

Works with **Doctrine ORM** for aggregate persistence, **Symfony Messenger Transport** for async message channels, and standard **Bundle configuration**. Ecotone auto-discovers your attributes in the `src` directory.

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
```

- [ ] **Step 2: Verify all internal links resolve**

Check that these paths exist:
- `modelling/command-handling/`
- `modelling/event-sourcing/`
- `modelling/business-workflows/`
- `modelling/recovering-tracing-and-monitoring/resiliency/`
- `modelling/microservices-php/`
- `messaging/multi-tenancy-support/`
- `modules/laravel/`
- `modules/symfony/`
- `modules/ecotone-lite/`
- `enterprise.md`

- [ ] **Step 3: Commit**

```bash
git add why-ecotone.md
git commit -m "Add Why Ecotone positioning page for architects and evaluators"
```

---

### Task 3: New Solutions Section — README and First 3 Pages

**Files:**
- Create: `solutions/README.md`
- Create: `solutions/scattered-application-logic.md`
- Create: `solutions/unreliable-async-processing.md`
- Create: `solutions/complex-business-processes.md`

- [ ] **Step 1: Create solutions directory**

```bash
mkdir -p solutions
```

- [ ] **Step 2: Create Solutions README**

Create `solutions/README.md`:

```markdown
---
description: Common challenges Ecotone solves for Laravel and Symfony developers
---

# Solutions

Every feature in Ecotone exists to solve a real problem that PHP developers face as their applications grow. If you recognize your situation below, click through to see how Ecotone addresses it.

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}
```

- [ ] **Step 3: Create Scattered Application Logic page**

Create `solutions/scattered-application-logic.md`:

```markdown
---
description: How to organize business logic with CQRS in Laravel and Symfony using Ecotone
---

# Scattered Application Logic

## The Problem You Recognize

Your application started clean, but as features grew, the boundaries blurred. Controllers handle business logic. Services read and write in the same method. Event listeners trigger side effects that nobody can trace.

In **Laravel**, you might have a 300-line Controller that validates input, queries the database, applies business rules, dispatches jobs, and returns a response — all in one method.

In **Symfony**, you might have a service class with 10 injected dependencies, where changing how orders are placed breaks the order listing page because both share the same service.

The symptoms are familiar:
- New developers take weeks to understand what happens when a user places an order
- Testing a single business rule requires setting up the entire framework
- A change in one area causes failures in unrelated features

## What the Industry Calls It

**CQRS — Command Query Responsibility Segregation.** Separate the code that changes state (commands) from the code that reads state (queries). Add event handlers for side effects. Each handler has one job.

## How Ecotone Solves It

With Ecotone, you organize your code around **Command Handlers**, **Query Handlers**, and **Event Handlers** — each responsible for exactly one thing. Ecotone wires them together automatically through PHP attributes:

```php
class OrderService
{
    #[CommandHandler]
    public function placeOrder(PlaceOrder $command): void
    {
        // Only handles placing the order — nothing else
    }

    #[QueryHandler("order.get")]
    public function getOrder(GetOrder $query): OrderView
    {
        // Only handles reading — no side effects
    }
}

class NotificationService
{
    #[EventHandler]
    public function whenOrderPlaced(OrderWasPlaced $event): void
    {
        // Reacts to the event — fully decoupled from order logic
    }
}
```

No base classes. No interfaces to implement. Your existing Laravel or Symfony services stay exactly where they are — you add attributes to give them clear responsibilities.

## Next Steps

- [CQRS Introduction — Commands](../modelling/command-handling/external-command-handlers/) — Learn how to define and dispatch commands
- [Query Handling](../modelling/command-handling/external-command-handlers/query-handling.md) — Separate your read models
- [Event Handling](../modelling/command-handling/external-command-handlers/event-handling.md) — React to domain events
- [Aggregate Introduction](../modelling/command-handling/state-stored-aggregate/) — Encapsulate business rules in a single place
```

- [ ] **Step 4: Create Unreliable Async Processing page**

Create `solutions/unreliable-async-processing.md`:

```markdown
---
description: How to build reliable async processing in Laravel and Symfony with Ecotone
---

# Unreliable Async Processing

## The Problem You Recognize

You added async processing to handle background work — sending emails, processing payments, syncing data. But now you have new problems:

- Failed jobs **disappear silently** or retry forever with no visibility
- You **can't replay** a failed message after fixing the bug — the data is gone
- A **duplicate webhook** triggers the same handler twice, leading to double charges or duplicate emails
- Going async **required touching every handler** — adding queue configuration, serialization, and retry logic to each one

In **Laravel**, you've scattered `dispatch()` calls and `ShouldQueue` implementations across your codebase. In **Symfony**, you've configured Messenger transports and retry strategies in YAML, but each handler still needs custom error handling.

## What the Industry Calls It

**Resilient Messaging** — a combination of patterns: automatic retries, error channels, dead letter queues, the outbox pattern for guaranteed delivery, and idempotency for deduplication.

## How Ecotone Solves It

With Ecotone, making a handler async is a single attribute. Resilience is built into the messaging layer — not bolted on per handler:

```php
// Make any handler async with one attribute
#[Asynchronous("notifications")]
#[EventHandler]
public function sendWelcomeEmail(UserRegistered $event): void
{
    // If this fails, Ecotone retries automatically
    // If it keeps failing, it goes to the dead letter queue
    // You can replay it after fixing the bug
}
```

Retries, error channels, and dead letter queues are configured once at the channel level — every handler on that channel gets production resilience automatically:

```php
// Deduplication prevents double-processing
#[ServiceContext]
public function messagingConfiguration(): array
{
    return [
        // All handlers on "notifications" channel get these settings
        SimpleMessageChannelBuilder::createQueueChannel("notifications"),
    ];
}
```

## Next Steps

- [Asynchronous Handling](../modelling/asynchronous-handling/) — Make handlers async with a single attribute
- [Retries](../modelling/recovering-tracing-and-monitoring/resiliency/retries.md) — Configure automatic retry strategies
- [Error Channel and Dead Letter](../modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/) — Store failed messages for replay
- [Outbox Pattern](../modelling/recovering-tracing-and-monitoring/resiliency/outbox-pattern.md) — Guarantee message delivery
- [Idempotency (Deduplication)](../modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md) — Prevent double-processing

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Command Bus Instant Retries](../modelling/recovering-tracing-and-monitoring/resiliency/retries.md#customized-instant-retries) for synchronous commands, [Command Bus Error Channel](../modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/#command-bus-error-channel) for centralized error routing, and [Gateway-Level Deduplication](../modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md#deduplication-with-command-bus) that protects every handler behind a bus automatically.
{% endhint %}
```

- [ ] **Step 5: Create Complex Business Processes page**

Create `solutions/complex-business-processes.md`:

```markdown
---
description: How to manage complex multi-step business workflows in PHP with Ecotone
---

# Complex Business Processes

## The Problem You Recognize

Your order fulfillment process spans 6 steps across 4 services. The subscription lifecycle involves payment processing, provisioning, notifications, and grace periods. User onboarding triggers a welcome email, account setup, and a follow-up sequence.

The logic for these processes is spread across:
- **Event listeners** that trigger other event listeners
- **Cron jobs** that check status flags
- **Database columns** like `is_processed`, `retry_count`, `step_completed_at`

Nobody can explain the full flow without reading all the code. Adding a step means editing multiple files. Reordering steps is risky. When something fails mid-process, recovery means manually updating database flags.

## What the Industry Calls It

**Sagas** (stateful workflows that remember where they are) and **Workflow Orchestration** (declarative step-by-step process definitions).

## How Ecotone Solves It

Ecotone provides three approaches depending on your complexity level:

**Simple linear workflows** — Chain handlers together with output channels:

```php
#[CommandHandler(
    routingKey: "order.place",
    outputChannelName: "order.verify_payment"
)]
public function placeOrder(PlaceOrder $command): OrderData
{
    // Step 1: Create the order, pass to next step
}

#[InternalHandler(
    inputChannelName: "order.verify_payment",
    outputChannelName: "order.ship"
)]
public function verifyPayment(OrderData $order): OrderData
{
    // Step 2: Verify payment, pass to shipping
}
```

**Stateful workflows** — Sagas remember state across long-running processes:

```php
#[Saga]
class OrderFulfillment
{
    #[Identifier]
    private string $orderId;
    private string $status;

    #[EventHandler]
    public static function start(OrderWasPlaced $event): self
    {
        // Begin the saga — tracks state across events
    }

    #[EventHandler]
    public function onPaymentReceived(PaymentReceived $event, CommandBus $bus): void
    {
        $this->status = 'paid';
        $bus->send(new ShipOrder($this->orderId));
    }
}
```

## Next Steps

- [Handler Chaining](../modelling/business-workflows/connecting-handlers-with-channels.md) — Simple linear workflows
- [Sagas](../modelling/business-workflows/sagas.md) — Stateful workflows that remember
- [Handling Failures](../modelling/business-workflows/handling-failures.md) — Recovery and compensation

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Orchestrators](../modelling/business-workflows/orchestrators.md) — declarative workflow automation where you define step sequences in one place, with each step independently testable and reusable. Dynamic step lists adapt to input data without touching step code.
{% endhint %}
```

- [ ] **Step 6: Commit**

```bash
git add solutions/
git commit -m "Add Solutions section with first 3 problem-oriented pages"
```

---

### Task 4: Solutions Section — Remaining 3 Pages

**Files:**
- Create: `solutions/audit-trail-and-state-rebuild.md`
- Create: `solutions/microservice-communication.md`
- Create: `solutions/php-for-enterprise-architecture.md`

- [ ] **Step 1: Create Audit Trail & State Rebuild page**

Create `solutions/audit-trail-and-state-rebuild.md`:

```markdown
---
description: How to implement Event Sourcing for audit trails and state rebuilds in PHP
---

# Audit Trail & State Rebuild

## The Problem You Recognize

A customer disputes a charge. Your support team asks "what exactly happened to this order?" The answer requires reading application logs, database timestamps, and hoping someone didn't overwrite the data.

Your read models need a schema change. You write a migration script, but there's no way to verify the migrated data is correct — the original events that created it are gone. You store the current state, but not how you got there.

The symptoms:
- **No history** — you know what the current price is, but not what it was yesterday
- **Risky migrations** — changing the read model means writing one-off scripts and praying
- **Compliance gaps** — auditors ask for a complete trail of changes and you can't provide one

## What the Industry Calls It

**Event Sourcing** — instead of storing the current state, store the sequence of events that led to it. Rebuild any view of the data by replaying events. Get a complete, immutable audit trail for free.

## How Ecotone Solves It

Ecotone provides Event Sourcing as a first-class feature with built-in projections. Your aggregate records events instead of mutating state:

```php
#[EventSourcingAggregate]
class Order
{
    #[Identifier]
    private string $orderId;

    #[CommandHandler]
    public static function place(PlaceOrder $command): array
    {
        return [new OrderWasPlaced($command->orderId, $command->items)];
    }

    #[EventSourcingHandler]
    public function onOrderPlaced(OrderWasPlaced $event): void
    {
        $this->orderId = $event->orderId;
    }
}
```

Build read models (projections) that can be rebuilt at any time from the event history:

```php
#[Projection("order_list", OrderStream::class)]
class OrderListProjection
{
    #[EventHandler]
    public function onOrderPlaced(OrderWasPlaced $event): void
    {
        // Build your read model — rebuildable from history
    }
}
```

Works with **Postgres, MySQL, and MariaDB** for event storage. Projections can write to any storage you choose.

## Next Steps

- [Event Sourcing Introduction](../modelling/event-sourcing/event-sourcing-introduction/) — How Event Sourced Aggregates work
- [Projections](../modelling/event-sourcing/setting-up-projections/) — Build and rebuild read models
- [Event Versioning](../modelling/event-sourcing/event-sourcing-introduction/event-versioning.md) — Evolve your events safely
- [Event Stream Persistence](../modelling/event-sourcing/event-sourcing-introduction/persistence-strategy.md) — Storage strategies and snapshots

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Partitioned Projections](../modelling/event-sourcing/setting-up-projections/scaling-and-advanced.md#partitioned-projections) for independent per-aggregate processing, [Async Backfill & Rebuild](../modelling/event-sourcing/setting-up-projections/backfill-and-rebuild.md) with parallel workers, and [Blue-Green Deployments](../modelling/event-sourcing/setting-up-projections/blue-green-deployments.md) for zero-downtime projection updates.
{% endhint %}
```

- [ ] **Step 2: Create Microservice Communication page**

Create `solutions/microservice-communication.md`:

```markdown
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
```

- [ ] **Step 3: Create PHP for Enterprise Architecture page**

Create `solutions/php-for-enterprise-architecture.md`:

```markdown
---
description: Enterprise architecture patterns in PHP - comparing Ecotone to Spring, Axon, NServiceBus
---

# PHP for Enterprise Architecture

## The Problem You Recognize

You're a technical lead or architect evaluating whether PHP can handle enterprise-grade architecture. Your team knows PHP well, but the business is growing — you need CQRS, Event Sourcing, distributed messaging, multi-tenancy, and production resilience.

The alternative is migrating to Java (Spring + Axon) or .NET (NServiceBus, MassTransit). That means retraining your team, rewriting your application, and losing PHP's development speed.

## PHP Has Grown Up

PHP is no longer just for simple web applications. Modern PHP (8.1+) has union types, enums, fibers, readonly properties, and first-class attributes. Frameworks like Laravel and Symfony provide the web layer. What was missing was the **enterprise messaging layer** — the equivalent of what Spring Integration and NServiceBus provide in their ecosystems.

Ecotone fills that gap. Built on the same [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) that underpin Spring Integration, NServiceBus, and Apache Camel, Ecotone brings production-grade enterprise patterns to PHP.

## How Ecotone Compares

| Capability | Java (Axon) | .NET (NServiceBus) | PHP (Ecotone) |
|-----------|-------------|---------------------|---------------|
| CQRS | Yes | Yes | Yes |
| Event Sourcing | Yes | Community | Yes |
| Sagas | Yes | Yes | Yes |
| Workflow Orchestration | Yes | Yes | Yes (Enterprise) |
| Distributed Messaging | Yes | Yes | Yes |
| Multi-Tenancy | Manual | Manual | Built-in |
| Message Broker Support | Kafka, RabbitMQ, etc. | RabbitMQ, Azure, etc. | RabbitMQ, Kafka, SQS, Redis |
| Observability | Micrometer | OpenTelemetry | OpenTelemetry |
| Testing Support | Axon Test Fixtures | NServiceBus Testing | Built-in Test Support |

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
```

- [ ] **Step 4: Commit**

```bash
git add solutions/
git commit -m "Add remaining 3 Solutions pages: audit trail, microservices, enterprise architecture"
```

---

### Task 5: Navigation Changes (SUMMARY.md)

**Files:**
- Modify: `SUMMARY.md`

- [ ] **Step 1: Add Why Ecotone and Solutions to SUMMARY.md**

Insert the following lines after line 1 (`* [About](README.md)`) and before line 2 (`* [Installation](install-php-service-bus.md)`):

```markdown
* [Why Ecotone?](why-ecotone.md)
* [Solutions](solutions/README.md)
  * [Scattered Application Logic](solutions/scattered-application-logic.md)
  * [Unreliable Async Processing](solutions/unreliable-async-processing.md)
  * [Complex Business Processes](solutions/complex-business-processes.md)
  * [Audit Trail & State Rebuild](solutions/audit-trail-and-state-rebuild.md)
  * [Microservice Communication](solutions/microservice-communication.md)
  * [PHP for Enterprise Architecture](solutions/php-for-enterprise-architecture.md)
```

The resulting top of SUMMARY.md should look like:

```markdown
# Table of contents

* [About](README.md)
* [Why Ecotone?](why-ecotone.md)
* [Solutions](solutions/README.md)
  * [Scattered Application Logic](solutions/scattered-application-logic.md)
  * [Unreliable Async Processing](solutions/unreliable-async-processing.md)
  * [Complex Business Processes](solutions/complex-business-processes.md)
  * [Audit Trail & State Rebuild](solutions/audit-trail-and-state-rebuild.md)
  * [Microservice Communication](solutions/microservice-communication.md)
  * [PHP for Enterprise Architecture](solutions/php-for-enterprise-architecture.md)
* [Installation](install-php-service-bus.md)
* [How to use](quick-start-php-ddd-cqrs-event-sourcing/README.md)
```

Everything below `* [Installation]` remains unchanged.

- [ ] **Step 2: Commit**

```bash
git add SUMMARY.md
git commit -m "Add Why Ecotone and Solutions section to navigation"
```

---

### Task 6: Reframe Feature Pages — Message Bus, CQRS, and Aggregates

**Files:**
- Modify: `modelling/command-handling/README.md` (insert problem header after line 5)
- Modify: `modelling/command-handling/state-stored-aggregate/README.md` (insert problem header after line 5)

- [ ] **Step 1: Add problem header to Message Bus and CQRS page**

In `modelling/command-handling/README.md`, insert the following after line 5 (`# Message Bus and CQRS`) and before line 7 (`In this chapter we will cover`):

```markdown

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

Your service classes mix reading and writing. A single change to how orders are placed breaks the order listing page. Business rules are scattered across controllers, listeners, and services — there's no clear boundary between "what changes state" and "what reads state."

## How Ecotone Solves It

Ecotone introduces **Command Handlers** for state changes, **Query Handlers** for reads, and **Event Handlers** for reactions. Each has a single responsibility, wired automatically through PHP attributes. No base classes, no framework coupling — just clear separation of concerns on top of your existing Laravel or Symfony application.

---

```

- [ ] **Step 2: Add problem header to Aggregates page**

In `modelling/command-handling/state-stored-aggregate/README.md`, insert the following after line 5 (`# Aggregate Introduction`) and before line 7 (`This chapter will cover`):

```markdown

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

Business rules are enforced in multiple places — a validation here, a check there. When rules change, you update three files and miss a fourth. There's no single source of truth for what an Order or User can do, and no guarantee that business invariants are always protected.

## How Ecotone Solves It

Ecotone's **Aggregates** encapsulate business rules in a single class. Commands are routed directly to the aggregate, which protects its own invariants. Ecotone handles loading and saving — you write business logic, not infrastructure code.

---

```

- [ ] **Step 3: Commit**

```bash
git add modelling/command-handling/README.md modelling/command-handling/state-stored-aggregate/README.md
git commit -m "Add problem-first headers to CQRS and Aggregates pages"
```

---

### Task 7: Reframe Feature Pages — Event Sourcing and Resiliency

**Files:**
- Modify: `modelling/event-sourcing/README.md` (rewrite — currently just links)
- Modify: `modelling/event-sourcing/event-sourcing-introduction/README.md` (insert problem header after line 5)
- Modify: `modelling/recovering-tracing-and-monitoring/resiliency/README.md` (rewrite — currently empty)

- [ ] **Step 1: Rewrite Event Sourcing section intro**

Replace the entire content of `modelling/event-sourcing/README.md` with:

```markdown
---
description: Event Sourcing PHP
---

# Event Sourcing

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

You store the current state of your entities, but not how they got there. When a customer disputes a charge, you can't answer "what exactly happened?" Rebuilding read models after a schema change means writing migration scripts by hand. Auditors ask for a complete trail of changes and you piece it together from application logs.

## How Ecotone Solves It

Ecotone provides **Event Sourcing** as a first-class feature. Instead of storing current state, you store the sequence of events that led to it. Rebuild any view of the data by replaying events. Get a complete, immutable audit trail automatically. Works with **Postgres**, **MySQL**, and **MariaDB** for event storage, with projections that can write to any storage you choose.

---

Read more in the following chapters.

## Materials

### Demo implementation

* [Implementing Event Sourcing Aggregates](https://github.com/ecotoneframework/quickstart-examples/tree/main/EventSourcing)
* [Emitting Events from Projections](https://github.com/ecotoneframework/quickstart-examples/tree/main/EmittingEventsFromProjection)
* [Working directly with Event Sourcing Aggregates](https://github.com/ecotoneframework/quickstart-examples/tree/main/WorkingWithAggregateDirectly)

### Links

* [Starting with Event Sourcing in PHP](https://blog.ecotone.tech/starting-with-event-sourcing-in-php/) \[Article]
* [Implementing Event Sourcing Application in 15 minutes](https://blog.ecotone.tech/implementing-event-sourcing-php-application-in-15-minutes/) \[Article]
```

- [ ] **Step 2: Add problem header to Event Sourcing Introduction page**

In `modelling/event-sourcing/event-sourcing-introduction/README.md`, insert the following after line 5 (`# Event Sourcing Introduction`) and before line 7 (`Before diving into`):

```markdown

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

You store the current state but not how you got there. When a customer disputes a charge, you can't answer "what exactly happened?" Rebuilding read models after a schema change means writing migration scripts by hand.

## How Ecotone Solves It

Ecotone's **Event Sourced Aggregates** store events instead of current state. Every state change is an immutable event in a stream. Projections rebuild read models from event history — change the schema, replay the events, get a correct read model.

---

```

- [ ] **Step 3: Rewrite Resiliency section intro**

Replace the entire content of `modelling/recovering-tracing-and-monitoring/resiliency/README.md` with:

```markdown
# Resiliency

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

A failed HTTP call crashes your handler. A duplicate webhook triggers double-processing. You've wrapped handlers in try/catch blocks and retry loops — each one slightly different. Error handling is scattered across your codebase with no consistent strategy.

## How Ecotone Solves It

Ecotone handles failures at the **messaging layer** — not per handler. Automatic retries, error channels, dead letter queues, the outbox pattern, and idempotency are configured once and apply to all handlers on a channel. When something fails, messages are preserved and can be replayed after the bug is fixed.

---

Explore the resiliency features:

- [Retries](retries.md) — Automatic retry strategies for transient failures
- [Error Channel and Dead Letter](error-channel-and-dead-letter/) — Store failed messages for later replay
- [Final Failure Strategy](final-failure-strategy.md) — What happens when all retries are exhausted
- [Idempotency (Deduplication)](idempotent-consumer-deduplication.md) — Prevent double-processing
- [Resilient Sending](resilient-sending.md) — Guaranteed delivery to async channels
- [Outbox Pattern](outbox-pattern.md) — Atomic message publishing with database transactions
- [Concurrency Handling](concurrency-handling.md) — Optimistic and pessimistic locking
```

- [ ] **Step 4: Commit**

```bash
git add modelling/event-sourcing/README.md modelling/event-sourcing/event-sourcing-introduction/README.md modelling/recovering-tracing-and-monitoring/resiliency/README.md
git commit -m "Add problem-first headers to Event Sourcing and Resiliency pages"
```

---

### Task 8: Reframe Feature Pages — Async, Workflows, Microservices

**Files:**
- Modify: `modelling/asynchronous-handling/README.md` (insert problem header after line 5)
- Modify: `modelling/business-workflows/README.md` (insert problem header before existing content)
- Modify: `modelling/microservices-php/README.md` (insert problem header after line 5)

- [ ] **Step 1: Add problem header to Async Handling page**

In `modelling/asynchronous-handling/README.md`, insert the following after line 5 (`# Asynchronous Handling and Scheduling`) and before line 7 (`{% content-ref`):

```markdown

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

You added async processing, but now you can't tell which messages are stuck, which failed silently, and which will retry forever. Going async required touching every handler — adding queue configuration, serialization logic, and retry strategies individually.

## How Ecotone Solves It

Ecotone makes any handler async with a single `#[Asynchronous]` attribute. Retries, error handling, and dead letter are configured at the channel level, not per handler. Switch between synchronous and asynchronous execution without changing your business code.

---

```

- [ ] **Step 2: Add problem header to Business Workflows page**

In `modelling/business-workflows/README.md`, insert the following after line 1 (`# Business Workflows`) and before line 3 (`Ecotone provides three`):

```markdown

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

Your order fulfillment process spans 6 steps across 4 services. The logic is spread across event listeners, cron jobs, and status flags. Nobody can explain the full flow without reading all the code. Adding or reordering steps risks breaking the entire process.

## How Ecotone Solves It

Ecotone provides three workflow approaches — from simple handler chaining to stateful sagas to declarative orchestrators. Each step is independently testable. The workflow definition lives in one place, not scattered across the codebase.

---

```

- [ ] **Step 3: Add problem header to Distributed Bus / Microservices page**

In `modelling/microservices-php/README.md`, insert the following after line 5 (`# Distributed Bus and Microservices`) and before line 7 (`Ecotone comes with Support`):

```markdown

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

You're calling other services via HTTP. When Service B is down, Service A fails too. You've built custom retry logic, custom serialization, and custom routing for each service pair. There's no event sharing — services can't subscribe to each other's events without point-to-point integrations.

## How Ecotone Solves It

Ecotone's **Distributed Bus** provides cross-service messaging through message brokers (RabbitMQ, SQS, Redis, Kafka). Services send commands and publish events to each other with guaranteed delivery. Swap transports without changing application code.

---

```

- [ ] **Step 4: Commit**

```bash
git add modelling/asynchronous-handling/README.md modelling/business-workflows/README.md modelling/microservices-php/README.md
git commit -m "Add problem-first headers to Async, Workflows, and Microservices pages"
```

---

### Task 9: Reframe Feature Pages — Multi-Tenancy and Testing

**Files:**
- Modify: `messaging/multi-tenancy-support/README.md` (insert problem header after line 5)
- Modify: `modelling/testing-support/README.md` (insert problem header after line 5)

- [ ] **Step 1: Add problem header to Multi-Tenancy page**

In `messaging/multi-tenancy-support/README.md`, insert the following after line 5 (`# Multi-Tenancy Support`) and before line 7 (`Ecotone provides support`):

```markdown

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

Adding a new tenant means configuring new queues, new database connections, and custom routing. A slow tenant's queue backlog delays everyone else's messages. Your multi-tenancy logic is tangled into business code instead of being handled at the infrastructure level.

## How Ecotone Solves It

Ecotone provides **built-in multi-tenancy** where the code you write is the same as in non-multi-tenant systems. Tenant context is propagated automatically through messages. Your business code doesn't need to know about tenants — Ecotone handles routing, isolation, and tenant switching at the messaging layer.

---

```

- [ ] **Step 2: Add problem header to Testing Support page**

In `modelling/testing-support/README.md`, insert the following after line 5 (`# Testing Support`) and before line 7 (`Ecotone provides comprehensive`):

```markdown

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

Testing your message handlers requires bootstrapping the entire framework, setting up queues, and hoping the async parts work. Unit testing a saga means mocking half the application. There's no way to test a message flow end-to-end without spinning up infrastructure.

## How Ecotone Solves It

Ecotone provides **in-memory testing** that lets you test message flows, aggregates, sagas, and event sourcing without external infrastructure. Send commands, verify events were published, check saga state — all in isolated unit tests.

---

```

- [ ] **Step 3: Commit**

```bash
git add messaging/multi-tenancy-support/README.md modelling/testing-support/README.md
git commit -m "Add problem-first headers to Multi-Tenancy and Testing pages"
```

---

### Task 10: Enterprise Page Enhancement

**Files:**
- Modify: `enterprise.md` (add positioning context and comparison table)

- [ ] **Step 1: Add positioning context to Enterprise page**

In `enterprise.md`, insert the following after line 1 (`# Enterprise`) and before line 3 (`## Ecotone Plans`):

```markdown

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

```

- [ ] **Step 2: Commit**

```bash
git add enterprise.md
git commit -m "Add positioning context and comparison table to Enterprise page"
```

---

### Task 11: Introduction Page Rewrite

**Files:**
- Modify: `modelling/message-driven-php-introduction.md` (restructure to lead with practical value)

- [ ] **Step 1: Rewrite the introduction page**

Replace the content of `modelling/message-driven-php-introduction.md` from line 1 through line 27 (up to and including `## Application level code`) with:

```markdown
---
description: Message Driven System with Domain Driven Design principles in PHP
---

# Introduction

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## What Ecotone Gives You

Ecotone is a messaging framework that brings enterprise architecture patterns to PHP. It provides the infrastructure for **CQRS**, **Event Sourcing**, **Sagas**, **Distributed Messaging**, and **Production Resilience** — so you write business logic, not boilerplate.

Everything in Ecotone is built around **Messages**. Commands express intentions ("place this order"), Events express facts ("order was placed"), and Queries express questions ("what are this user's orders?"). This isn't just a naming convention — it's the architectural foundation that enables async processing, resilience, workflows, and distributed systems.

## Built on Enterprise Integration Patterns

Ecotone is built on [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) — the same foundation that powers Spring Integration (Java), NServiceBus (.NET), and Apache Camel. Communication between objects happens through **Message Channels** — pipes where one side sends messages and the other consumes them.

<figure><img src="../.gitbook/assets/communication.png" alt=""><figcaption><p>Communication between Objects using Messages</p></figcaption></figure>

Because communication goes through Message Channels, switching from synchronous to asynchronous, or from one message broker to another, doesn't affect your business code. You change the channel configuration — your handlers stay the same.

## Application level code

```

Keep everything from line 28 onwards (starting with `Ecotone provides different levels of abstractions`) unchanged.

- [ ] **Step 2: Commit**

```bash
git add modelling/message-driven-php-introduction.md
git commit -m "Rewrite Introduction to lead with practical value, keep technical depth"
```

---

### Task 12: Module Pages Enhancement

**Files:**
- Modify: `modules/laravel/README.md` (add positioning opening)
- Modify: `modules/symfony/README.md` (add positioning opening)

- [ ] **Step 1: Add positioning to Laravel module page**

In `modules/laravel/README.md`, insert the following after line 5 (`# Laravel`) and before line 7 (`## Installation`):

```markdown

Ecotone integrates with your existing Laravel application — **Eloquent** for aggregate persistence, **Laravel Queues** for async message channels, and **Laravel Octane** for high-performance scenarios. You keep Laravel's developer experience and add enterprise messaging architecture.

```

- [ ] **Step 2: Add positioning to Symfony module page**

In `modules/symfony/README.md`, insert the following after line 5 (`# Symfony`) and before line 7 (`## Installation`):

```markdown

Ecotone builds on top of your Symfony application — **Doctrine ORM** for aggregate persistence, **Symfony Messenger Transport** for async message channels, and standard **Bundle configuration**. Everything you know stays, enterprise patterns get added.

```

- [ ] **Step 3: Commit**

```bash
git add modules/laravel/README.md modules/symfony/README.md
git commit -m "Add enterprise positioning to Laravel and Symfony module pages"
```

---

### Task 13: Quick Start Section Enhancement

**Files:**
- Modify: `quick-start-php-ddd-cqrs-event-sourcing/php-cqrs.md` (add problem framing)
- Modify: `quick-start-php-ddd-cqrs-event-sourcing/asynchronous-php.md` (add problem framing)
- Modify: `quick-start-php-ddd-cqrs-event-sourcing/event-sourcing-php.md` (add problem framing)
- Modify: `quick-start-php-ddd-cqrs-event-sourcing/resiliency-and-error-handling.md` (add problem framing)
- Modify: `quick-start-php-ddd-cqrs-event-sourcing/microservices-php.md` (add problem framing)

- [ ] **Step 1: Read the quick start files to find insertion points**

Read the first 10 lines of each file to identify the heading line after which to insert the problem framing.

- [ ] **Step 2: Add problem framing to CQRS quick start**

In `quick-start-php-ddd-cqrs-event-sourcing/php-cqrs.md`, insert after the `# CQRS PHP` heading:

```markdown

Separate the code that changes state from the code that reads it — clear command and query handlers with zero boilerplate.

```

- [ ] **Step 3: Add problem framing to Async quick start**

In `quick-start-php-ddd-cqrs-event-sourcing/asynchronous-php.md`, insert after the `# Asynchronous PHP` heading:

```markdown

Make any handler async with a single attribute — retries, error handling, and dead letter included automatically.

```

- [ ] **Step 4: Add problem framing to Event Sourcing quick start**

In `quick-start-php-ddd-cqrs-event-sourcing/event-sourcing-php.md`, insert after the heading:

```markdown

Store events instead of current state — get a full audit trail and rebuildable read models for free.

```

- [ ] **Step 5: Add problem framing to Resiliency quick start**

In `quick-start-php-ddd-cqrs-event-sourcing/resiliency-and-error-handling.md`, insert after the heading:

```markdown

Automatic retries, dead letter queues, and message replay — production resilience without per-handler boilerplate.

```

- [ ] **Step 6: Add problem framing to Microservices quick start**

In `quick-start-php-ddd-cqrs-event-sourcing/microservices-php.md`, insert after the heading:

```markdown

Cross-service messaging with guaranteed delivery — send commands and share events between PHP services.

```

- [ ] **Step 7: Commit**

```bash
git add quick-start-php-ddd-cqrs-event-sourcing/
git commit -m "Add one-sentence problem framing to quick start pages"
```

---

### Task 14: Final Review and Verification

- [ ] **Step 1: Verify all new files exist**

```bash
ls -la why-ecotone.md solutions/
```

Expected: `why-ecotone.md` exists, `solutions/` contains 7 files (README.md + 6 solution pages).

- [ ] **Step 2: Verify SUMMARY.md has correct structure**

Read SUMMARY.md and verify:
- "Why Ecotone?" appears after "About"
- "Solutions" section with 6 sub-pages appears before "Installation"
- All paths in SUMMARY.md point to files that exist

- [ ] **Step 3: Verify all internal links in new pages**

Spot-check that the relative links in `solutions/` pages point to existing files:
```bash
# Check a sample of linked files exist
ls modelling/command-handling/external-command-handlers/
ls modelling/asynchronous-handling/
ls modelling/business-workflows/
ls modelling/event-sourcing/event-sourcing-introduction/
ls modelling/microservices-php/distributed-bus/
ls modelling/recovering-tracing-and-monitoring/resiliency/
```

- [ ] **Step 4: Review git log for clean commit history**

```bash
git log --oneline -15
```

Expected: Clean sequence of commits, one per task, with descriptive messages.
