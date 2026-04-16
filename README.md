---
description: Ecotone — The enterprise architecture layer for Laravel and Symfony
---

# About

<figure><img src=".gitbook/assets/ecotone_logo_no_background (1).png" alt="" width="563"><figcaption></figcaption></figure>

## Ecotone extends your existing Laravel and Symfony application with the enterprise architecture layer

One Composer package adds CQRS, Event Sourcing, Workflows, and production resilience to your codebase. No framework change. No base classes. Just PHP attributes on your existing code.

```bash
composer require ecotone/laravel    # or ecotone/symfony-bundle
```

---

## See what it looks like

```php
class OrderService
{
    #[CommandHandler] 
    public function placeOrder(PlaceOrder $command, EventBus $eventBus): void
    {
        // your business logic
        $eventBus->publish(new OrderWasPlaced($command->orderId));
    }

    #[QueryHandler('order.getStatus')]
    public function getStatus(string $orderId): string
    {
        return $this->orders[$orderId]->status;
    }
}

class NotificationService
{
    #[Asynchronous('notifications')]
    #[EventHandler]  
    public function whenOrderPlaced(OrderWasPlaced $event, NotificationSender $sender): void
    {
        $sender->sendOrderConfirmation($event->orderId);
    }
}
```

**That's the entire setup.** No bus configuration. No handler registration. No retry config. No serialization wiring. Ecotone reads your attributes and handles the rest:

* **Command and Query Bus** — wired automatically from your `#[CommandHandler]` and `#[QueryHandler]` attributes
* **Event routing** — `NotificationService` subscribes to `OrderWasPlaced` without any manual wiring
* **Async execution** — `#[Asynchronous('notifications')]` routes to RabbitMQ, SQS, Kafka, or DBAL — your choice of transport
* **Failure isolation** — each event handler gets its own copy of the message, so one handler's failure never blocks another
* **Retries and dead letter** — failed messages retry automatically, permanently failed ones go to a [dead letter queue](modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/) you can inspect and replay
* **Tracing** — [OpenTelemetry integration](modelling/recovering-tracing-and-monitoring/) traces every message across sync and async flows

### Test exactly the flow you care about

Extract a specific flow and test it in isolation — only the services you need:

```php
$ecotone = EcotoneLite::bootstrapFlowTesting([OrderService::class]);

$ecotone->sendCommand(new PlaceOrder('order-1'));

$this->assertEquals('placed', $ecotone->sendQueryWithRouting('order.getStatus', 'order-1'));
```

Only `OrderService` is loaded. No notifications, no other handlers — just the flow you're verifying.

Now bring in the full async flow. Enable an in-memory channel and run it within the same test process:

```php
$notifier = new InMemoryNotificationSender();

$ecotone = EcotoneLite::bootstrapFlowTesting(
    [OrderService::class, NotificationService::class],
    [NotificationSender::class => $notifier],
    enableAsynchronousProcessing: [
        SimpleMessageChannelBuilder::createQueueChannel('notifications')
    ]
);

$ecotone
    ->sendCommand(new PlaceOrder('order-1'))
    ->run('notifications');

$this->assertEquals(['order-1'], $notifier->getSentOrderConfirmations());
```

`->run('notifications')` processes messages from the in-memory queue — right in the same process. The async handler executes deterministically, no timing issues, no polling, no external broker.

**The key:** swap the in-memory channel for [DBAL](modules/dbal-support/), [RabbitMQ](modules/amqp-support-rabbitmq/), or [Kafka](modules/kafka-support/) to test what runs in production — the test stays the same. Ecotone runs the consumer within the same process, so switching transports never changes how you test. The ease of in-memory testing [stays with you](modelling/testing-support/) no matter what backs your production system.

---

## What changes in your daily work

### Business logic is the only code you write

No command bus configuration. No handler registration. No message serialization setup. You write a PHP class with an attribute, and Ecotone wires the bus, the routing, the serialization, and the async transport. Your code stays focused on what your application actually does — your domain.

### Going async never means rewriting handlers

Add `#[Asynchronous('channel')]` to any handler. The handler code stays identical. Switch from synchronous to [RabbitMQ](modules/amqp-support-rabbitmq/) to [SQS](modules/sqs-support/) to [Kafka](modules/kafka-support/) by changing one line of configuration. Your business logic never knows the difference.

### Failed messages don't disappear

Every failed message is captured in a [dead letter queue](modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/). You see what failed, the full exception, and the original message. [Replay it](modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/dbal-dead-letter.md) with one command. And can be combined with inbuilt Outbox pattern to ensure full consistency. No more silent failures. No more guessing what happened to that order at 3am.

### Complex workflows live in one place

A multi-step business process — order placement, payment, shipping, notification — doesn't need to be scattered across event listeners, cron jobs, and database flags. Ecotone gives you [Sagas](modelling/business-workflows/sagas.md) for stateful workflows, [handler chaining](modelling/business-workflows/connecting-handlers-with-channels.md) for linear pipelines, and [Orchestrators](modelling/business-workflows/orchestrators.md) for declarative process control. The entire business flow is readable in one class.

### Your codebase tells the story of your business

When a new developer opens your code, they see `PlaceOrder`, `OrderWasPlaced`, `ShipOrder` — not `AbstractMessageBusHandlerFactory`. Ecotone keeps your domain clean: no base classes to extend, no framework interfaces to implement, no infrastructure leaking into your business logic. Just [plain PHP objects](modelling/command-handling/) with attributes that declare their intent.

---

## AI-ready by design

Ecotone's declarative, attribute-based architecture is inherently friendly to AI code generators. When your AI assistant works with Ecotone code, two things happen:

**Less context needed, less code generated.** A command handler with `#[CommandHandler]` and `#[Asynchronous('orders')]` tells the full story in two attributes — no bus configuration files, no handler registration, no retry setup to feed into the AI's context window. The input is smaller because there's less infrastructure to read, and the output is smaller because there's less boilerplate to generate. That means lower token cost and faster iteration cycles.

**AI that knows Ecotone.** Your AI assistant can work with Ecotone out of the box:

* **[Agentic Skills](other/ai-integration.md)** — 17 ready-to-use skills that teach any coding agent how to correctly write handlers, aggregates, sagas, projections, tests, and more. Install with one command and your AI generates idiomatic Ecotone code from the start.
* **[MCP Server](other/ai-integration.md)** — Direct access to Ecotone documentation for any AI assistant that supports Model Context Protocol — Claude Code, Cursor, Windsurf, GitHub Copilot, and others.
* **[llms.txt](other/ai-integration.md)** — AI-optimized documentation files that give any LLM instant context about Ecotone's API and patterns.

**Testing that AI can actually run.** Ecotone's [testing support](modelling/testing-support/) runs async flows synchronously within the same process — no worker processes to spawn, no message brokers to configure, no timing to get right. This matters for AI-assisted development: even the most complex flows involving async handlers, sagas, and event-driven projections can be tested with a simple `->sendCommand()` and `->run()` call. Your coding agent can write and verify tests for async workflows without getting confused by infrastructure setup or hallucinating non-existent test utilities.

The result: your AI assistant writes correct Ecotone code faster, with less back-and-forth, because the framework's declarative design means there's simply less to get wrong — and it can verify its own work by running the tests.

---

## The full capability set

| Capability | What it gives you | Learn more |
|---|---|---|
| **CQRS** | Separate command and query handlers. Clean responsibility boundaries. Automatic bus wiring. | [Command Handling](modelling/command-handling/) |
| **Event Sourcing** | Store events instead of state. Full audit trail. Rebuild read models anytime. Time travel and replay. | [Event Sourcing](modelling/event-sourcing/) |
| **Workflows & Sagas** | Orchestrate multi-step business processes. Stateful workflows with compensation logic. | [Business Workflows](modelling/business-workflows/) |
| **Async Messaging** | RabbitMQ, Kafka, SQS, Redis, DBAL. One attribute to go async. Swap transports without code changes. | [Asynchronous Handling](modelling/asynchronous-handling/) |
| **Production Resilience** | Automatic retries, dead letter queues, outbox pattern, message deduplication, failure isolation. | [Resiliency](modelling/recovering-tracing-and-monitoring/resiliency/) |
| **Domain-Driven Design** | Aggregates, domain events, bounded contexts. Pure PHP objects with no framework coupling. | [Aggregates](modelling/command-handling/state-stored-aggregate/) |
| **Distributed Bus** | Cross-service messaging. Share events and commands between microservices with guaranteed delivery. | [Microservices](modelling/microservices-php/) |
| **Multi-Tenancy** | Tenant-isolated processing, projections, and event streams. Built in, not bolted on. | [Multi-Tenancy](messaging/multi-tenancy-support/) |
| **Observability** | OpenTelemetry integration. Trace every message — sync or async — across your entire system. | [Monitoring](modelling/recovering-tracing-and-monitoring/) |
| **Interceptors** | Cross-cutting concerns — authorization, logging, transactions — applied declaratively via attributes. | [Interceptors](modelling/extending-messaging-middlewares/interceptors/) |

---

## The enterprise gap in PHP, closed

Every mature ecosystem has an enterprise architecture layer on top of its web framework:

| Ecosystem | Web Framework | Enterprise Architecture Layer |
|---|---|---|
| **Java** | Spring Boot | Spring Integration + Axon Framework |
| **.NET** | ASP.NET | NServiceBus / MassTransit |
| **PHP** | Laravel / Symfony | **Ecotone** |

Ecotone is built on the same foundation — [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) — that powers Spring Integration, NServiceBus, and Apache Camel. In active development since 2017 and used in production by teams running multi-tenant, event-sourced systems at scale, Ecotone brings the same patterns that run banking, logistics, and telecom systems in Java and .NET to PHP.

This isn't about PHP catching up. It's about your team using proven architecture patterns — with the development speed that PHP gives you — without giving up the ecosystem you already know.

[Read more: Why Ecotone?](why-ecotone.md)

---

## Start with your framework

**Laravel** — Keep Eloquent, Laravel Queues, and Octane. Add what Laravel doesn't have: Event Sourcing, Sagas, failure isolation per handler, synchronous testing of async flows, and outbox pattern.\
`composer require ecotone/laravel`\
→ [Laravel Quick Start](quick-start-php-ddd-cqrs-event-sourcing/laravel-ddd-cqrs-demo-application.md) · [Laravel Module docs](modules/laravel/)

**Symfony** — Symfony Messenger gives you a message bus. Ecotone gives you the complete architecture: Event Sourcing with projections, Sagas for stateful workflows, per-handler failure isolation, dead letter queue with replay, and synchronous testing of async flows.\
`composer require ecotone/symfony-bundle`\
→ [Symfony Quick Start](quick-start-php-ddd-cqrs-event-sourcing/symfony-ddd-cqrs-demo-application/) · [Symfony Module docs](modules/symfony/)

**Any PHP framework** — Ecotone Lite works with any PSR-11 compatible container.\
`composer require ecotone/lite-application`\
→ [Ecotone Lite docs](modules/ecotone-lite/)

---

**Try it in one handler.** You don't need to migrate your application. Install Ecotone, add an attribute to one handler, and see what happens. If you like what you see, add more. If you don't — remove the package. Zero commitment.

* [Install](install-php-service-bus.md) — Setup guide for any framework
* [Learn by example](quick-start-php-ddd-cqrs-event-sourcing/) — Send your first command in 5 minutes
* [Go through tutorial](tutorial-php-ddd-cqrs-event-sourcing/) — Build a complete messaging flow step by step
* [Workshops, Support, Consultancy](other/contact-workshops-and-support.md) — Hands-on training for your team

{% hint style="info" %}
The full CQRS, Event Sourcing, and Workflow feature set is [free and open source](enterprise.md) under the Apache 2.0 License. [Enterprise features](enterprise.md) are available for teams that need advanced scaling, distributed bus with service map, orchestrators, and production-grade Kafka integration.
{% endhint %}

{% hint style="success" %}
Join [Ecotone's Community Channel](https://discord.gg/GwM2BSuXeg) — ask questions and share what you're building.
{% endhint %}
