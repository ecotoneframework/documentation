---
description: >-
  Ecotone — the PHP architecture layer that grows with your system, without
  rewrites
---

# About

<figure><img src=".gitbook/assets/ecotone_logo_no_background.png" alt="" width="563"><figcaption></figcaption></figure>

## The PHP architecture layer that grows with your system — without rewrites

You started clean — controllers, services, jobs. Eighteen months later, business logic is scattered across listeners, the queue retries handlers that already succeeded, the Stripe webhook double-charges customers because nothing dedupes it, and adding event sourcing means swapping libraries. Ecotone is built around a different growth path: **one package** that takes you from `#[CommandHandler]` on day one to event sourcing, sagas, outbox, and distributed messaging at scale — same codebase, no forced migrations between growth stages.

Works on the Laravel or Symfony you already run.

```bash
composer require ecotone/laravel    # or ecotone/symfony-bundle
```

**Install today. Write a command handler in the next five minutes. Add event sourcing, workflows, asynchronous processing when need — by writing new classes, not swapping libraries.**

***

## Get Started Smoothly

You start here on **day one**:

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

* **Command, Query, and Event Bus** — wired automatically from your `#[CommandHandler]`, `#[QueryHandler]`, and `#[EventHandler]` attributes
* **Event routing** — `NotificationService` subscribes to `OrderWasPlaced` without any manual wiring
* **Async execution** — `#[Asynchronous('notifications')]` routes to RabbitMQ, SQS, Kafka, or DBAL — your choice of transport
* **Failure isolation** — each event handler receives its own copy of the message; one handler's failure never affects another
* **Retries and dead letter** — failed messages retry automatically, permanently failed ones go to a [dead letter queue](modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/) you can inspect and replay
* **Tracing** — [OpenTelemetry integration](modelling/recovering-tracing-and-monitoring/) traces every message across sync and async flows

### Test exactly the flow you care about

Extract a specific flow and test it in isolation — only the services you need:

```php
$ecotone = EcotoneLite::bootstrapFlowTesting([OrderService::class]);

$ecotone->sendCommand(new PlaceOrder('order-1'));

$this->assertEquals('placed', $ecotone->sendQueryWithRouting('order.getStatus', 'order-1'));
```

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

**The key:** swap the in-memory channel for [DBAL](modules/dbal-support/), [RabbitMQ](modules/amqp-support-rabbitmq/), or [Kafka](modules/kafka-support/) to test what runs in production — the test stays the same. The ease of in-memory testing [stays with you](modelling/testing-support/) no matter what backs your production system.

***

## The growth ladder

```
   Day 1               Week 1                 Month 3                Year 1+
──────────────────   ───────────────────   ────────────────────   ──────────────────────
#[CommandHandler]    + #[Asynchronous]     + #[Saga]              + #[EventSourcing
#[EventHandler]      + #[Interceptor]      + Workflows              Aggregate]
#[QueryHandler]      + Retries / DLQ       + transactional        + #[Projection]
                                             outbox               + #[DistributedBus]
                                                                  + Multi Tenant Channels
──────────────────   ───────────────────   ────────────────────   ──────────────────────
  Familiar handlers.   Async with            Stateful workflows.    Event sourcing &
  Five-minute start.   resilience built in.  Outbox-consistent.     distributed bus.

          Same classes. Same codebase. Same team. No forced migration between stages.
```

Every other PHP alternative forces you to re-decide architecture at each column break — swap libraries, add glue, or stitch together a multi-package stack. No other _single_ PHP package spans the full set of growth stages.

***

## Why companies pick Ecotone

#### 1. Composable with alternatives. Complete standalone.

Ecotone doesn't have to be the entire stack. It composes with the other libraries above when a team already has a preferred tool for one layer:

* **Ecotone as the orchestration / saga / outbox layer**
* **Ecotone as the CQRS, workflows, and asynchronous communication layer**
* **Ecotone as the Event Sourcing layer**

**Ecotone is the only toolkit on this list that runs fully standalone.** Every other option requires you to integrate something for the layers it does not cover. Ecotone covers every layer itself when you want it to, and composes with alternatives when you prefer a specific tool for one piece. One package to upgrade, one model to teach.

#### 2. True per-handler failure isolation — a copy of the message to every handler

Ecotone delivers per-handler failure isolation by dispatching a **copy of the message to every handler**. Each handler processes its own message, retries independently, and fails independently. A failure in one handler cannot affect, block, or re-trigger any sibling handler, because they are never sharing a message envelope in the first place.

This is the structural difference from Symfony Messenger: Messenger dispatches a single envelope through multiple handlers; per-handler failure isolation is not a property it provides, and the community workarounds (per-handler transports, idempotency keys, custom dedup middleware) reduce the problem without solving it at the framework level.

Pair that with a first-class transactional outbox and deduplication, and the three patterns you would otherwise assemble from idempotency middleware + a community outbox package + a custom dedup layer become default behavior.

#### 3. Production resilience is the default, not assembly

* **Transactional outbox** — business write and message publish in one atomic commit. No "we published but the DB rolled back" ghost events. No "we committed but the publish crashed" lost events.
* **Dead letter queue** — failed messages captured with full context, replayable with one command.
* **Deduplication** — built in, attribute-driven.
* **OpenTelemetry tracing** — every message traced across sync and async hops.
* **Retries with configurable backoff** — per channel, no custom wiring.

#### 4. Framework-portable business code

Same aggregates, handlers, sagas, and projections run on Laravel, Symfony, or any PSR-11 container. A team that starts on Laravel and later consolidates on Symfony (or vice versa) does not rewrite business logic. For companies hiring from both PHP pools, handling acquisitions, or wanting long-term framework independence, this removes a category of strategic risk.

No other option on this list offers framework portability.

#### 5. Battle-tested patterns, not invented ones

Ecotone is built on Enterprise Integration Patterns — the same foundation behind Spring Integration, Axon Framework (Java), NServiceBus and MassTransit (.NET), and Apache Camel. These patterns run banking, telecom, and logistics systems in production at scale, measured in decades.

Ecotone brings them to PHP as attribute-driven code on your existing Laravel or Symfony application. Your team writes POPOs; Ecotone applies the patterns.

#### 6. Composable messaging — pipe, route, split, filter, transform on the fly

The EIP foundation is not a lineage — it is a working catalogue of composable primitives you write as attributes:

* **Pipe handlers together** — output of one handler becomes input of the next, no `Bus::chain` glue code.
* **Content-based routing** — route the same message to different handlers based on its payload or headers. VIP orders go through the premium path, bulk orders through the batch path — no `if` statements in your domain code.
* **Splitters** — one `OrderPlaced` event with ten line items fans out into ten per-item fulfillment messages, processed in parallel, each independently retryable.
* **Filters and transformers** — reject, enrich, or reshape messages declaratively.

The problem these primitives solve is complex integration flows that usually collapse into unmaintainable conditional dispatch code. In Ecotone, each primitive is an attribute on a handler or channel. The flow is readable in one place, the failure modes are first-class, and adding a new routing rule is an attribute, not a refactor.

**See it built end-to-end:** [Composing Building Blocks](composing-building-blocks.md) — a walk-through of an Order Fulfillment system that uses every primitive above, with side-by-side Symfony Messenger and Laravel Queues comparisons.

#### 7. Multi-tenancy down to the message channel

Multi-tenancy in Ecotone is not just isolated storage. It extends through the messaging topology:

* **Tenant-isolated event streams** — each tenant's events live in their own stream. No cross-tenant replay risk, no cross-tenant projection leakage.
* **Tenant-routed message channels** — messages are automatically routed to per-tenant queues based on their metadata. One handler, many tenants, no cross-contamination.
* **Priority routing by customer status** — a VIP customer's request is dispatched to a fast queue and processed with higher priority; a free-tier request goes through the standard queue. Same handler code, declarative routing.
* **Tenant-aware projections** — each tenant gets its own read model, rebuildable in isolation.

This is multi-tenancy as an architectural property of the message bus, not a WHERE clause in your queries.

#### 8. Endpoint-ID routing — refactor without fear, deserialize only when needed

Ecotone routes messages by **endpoint ID**, not by class name. Three direct consequences:

* **Rename classes, move handlers, change namespaces — nothing breaks.** The endpoint ID is the contract. You can move a command handler from one module to another, rename the command class itself, refactor the entire package structure, and every queued message in flight will still be routed to the right handler.
* **Deserialize only when needed.** A message passing through a router or pipe is not deserialized until it reaches its actual endpoint. Messages that do not need processing at a given node stay serialized — meaningful throughput gains on high-fanout and pass-through workloads.
* **Protocol-stable integration between services.** Published messages stay compatible with consumers as your internal code structure evolves. The fully-qualified class name is not part of the wire contract; the endpoint ID is.

#### 9. Compose patterns without glue code — the attribute is the wiring

The reason CQRS, event sourcing, sagas, and projections compose seamlessly is that **they are all handlers on the same messaging foundation**. There is no wiring layer, no event-to-handler mapping configuration, no container registration, no factory class.

```php
#[Aggregate]
class Order {
    #[CommandHandler]                           // CommandBus routes here
    public static function place(PlaceOrder $command): self {
        $order = new self(/* ... */);
        $order->recordThat(new OrderWasPlaced(/* ... */));  // publishes event
        return $order;
    }
}

#[Saga]
class OrderFulfillmentProcess {
    #[EventHandler]                             // subscribes to OrderWasPlaced
    public function onOrderPlaced(OrderWasPlaced $event): void { /* ... */ }
}

#[Projection(name: 'orders_read_model')]
class OrdersProjection {
    #[EventHandler]                             // same event, different handler, own message copy
    public function whenOrderPlaced(OrderWasPlaced $event): void { /* ... */ }
}

class NotificationService {
    #[Asynchronous('notifications')]
    #[EventHandler]                             // same event again, async, independent retry
    public function notify(OrderWasPlaced $event): void { /* ... */ }
}
```

Four different building blocks — Aggregate, Saga, Projection, async event handler — subscribing to the same event. There is no `EventSubscriberInterface` list to register, no saga-to-event mapping file, no projection orchestrator. Each subscriber receives its own copy of the message, retries independently, and lives its own lifecycle.

This is what "one toolkit, every pattern, no glue" means in practice.

***

## Trusted in regulated and high-stakes production

Ecotone runs in production at:

* **Payment gateways** — where retried handlers cannot double-charge, and the outbox must guarantee that every committed transaction produces exactly one downstream message.
* **Credit card systems** — where transaction loss is catastrophic, and every state change must be auditable and replayable.
* **Certification authorities** — whose entire business depends on reconstructible, tamper-evident audit trails; the event log is the audit log.
* **E-commerce platforms** — orchestrating order → payment → fulfillment → notification as declarative sagas with compensation.
* **Public transportation subscription systems** — managing nationwide transit subscriptions (create, renew, terminate) with distributed bus integration to Java and PHP services over Kafka.
* **Two-sided marketplaces** — coordinating customer orders, provider subscriptions, lead distribution, and B2B enterprise partnerships on one event-driven backbone.

The features on the capability matrix below are not hypothetical. They run in systems where failure is either regulated (audit, payments), expensive (double-charges, lost deliveries), or public (transit, marketplaces).

***

## The Capability Matrix

| Capability                                                                                                                                                                                                                           | Included out of the box                                                                                                                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Command / Query / Event bus**                                                                                                                                                                                                      | Yes — auto-wired from attributes                                                                                                                                                                                        |
| **Per-handler failure isolation**                                                                                                                                                                                                    | Yes — copy of the message per handler, structural not workaround                                                                                                                                                        |
| **Event Sourcing**                                                                                                                                                                                                                   | Yes — aggregates, projections, replay, snapshotting                                                                                                                                                                     |
| **Partitioned projections** (parallel rebuild across workers, by aggregate / tenant / hash)                                                                                                                                          | Yes                                                                                                                                                                                                                     |
| **Streaming projections** (durable per-projector cursor, push-based catch-up)                                                                                                                                                        | Yes                                                                                                                                                                                                                     |
| **Non-blocking projection rebuild** (blue-green + concurrent async backfill across workers, scales to millions of events)                                                                                                            | Yes — `#[ProjectionDeployment]` + `#[ProjectionBackfill]` + aggregate-ID partitioning                                                                                                                                   |
| **Partition-scoped rebuild** (rebuild a single aggregate's data in milliseconds while other partitions keep serving reads)                                                                                                           | Yes — `#[PartitionAggregateId]` in `#[ProjectionReset]`; effectively zero downtime even during full rebuilds                                                                                                            |
| **Self-healing projections** (trigger-based execution — projection always reads from Event Store at its last committed position; fix-deploy recovery, no manual reset or backfill script)                                            | Yes — default behavior; events never lost after a crash                                                                                                                                                                 |
| **Gap detection** (sequence gaps from concurrent commits are tracked and retried; events never silently skipped)                                                                                                                     | Yes — enabled by default, non-blocking, bounded by `maxGapOffset` and `gapTimeout`                                                                                                                                      |
| **Batched projection persistence** (accumulate state across events via `#[ProjectionState]`; single `#[ProjectionFlush]` per batch; automatic Doctrine EntityManager flush + clear prevents OOM during multi-million-event catch-up) | Yes — `#[ProjectionExecution(eventLoadingBatchSize: N)]`                                                                                                                                                                |
| **Projection-emitted events** (downstream eventual-consistency notifications via `EventStreamEmitter`)                                                                                                                               | Yes — emitted events fan out through the event bus to sagas, handlers, and other projections; **automatic emission suppression during rebuild** so downstream consumers aren't flooded with duplicate historical events |
| **Sagas** (stateful long-running workflows)                                                                                                                                                                                          | Yes                                                                                                                                                                                                                     |
| **Handler chaining workflows** (stateless pipe-and-filter)                                                                                                                                                                           | Yes                                                                                                                                                                                                                     |
| **Content-based routing** (route by payload or headers)                                                                                                                                                                              | Yes                                                                                                                                                                                                                     |
| **Splitters** (fan-out)                                                                                                                                                                                                              | Yes                                                                                                                                                                                                                     |
| **Filters, transformers, enrichers**                                                                                                                                                                                                 | Yes — EIP primitives as attributes                                                                                                                                                                                      |
| **Priority** (sync ordering + native broker priority for async)                                                                                                                                                                      | Yes — one attribute, uniform across Event Handlers, Projections, Sagas                                                                                                                                                  |
| **Custom buses per use case**                                                                                                                                                                                                        | Yes — extend `CommandBus` / `EventBus` / `QueryBus`, attach `#[Deduplicated]`, `#[ErrorChannel]`, `#[InstantRetry]` directly to the interface declaration                                                               |
| **Endpoint-ID message routing** (class-name-independent)                                                                                                                                                                             | Yes — refactor-safe, lazy deserialization                                                                                                                                                                               |
| **Transactional outbox**                                                                                                                                                                                                             | Yes — DBAL + per-transport                                                                                                                                                                                              |
| **Dead letter queue + replay**                                                                                                                                                                                                       | Yes                                                                                                                                                                                                                     |
| **Retries with configurable backoff**                                                                                                                                                                                                | Yes                                                                                                                                                                                                                     |
| **Deduplication**                                                                                                                                                                                                                    | Yes                                                                                                                                                                                                                     |
| **Async transports**                                                                                                                                                                                                                 | RabbitMQ, Kafka, SQS, Redis, DBAL                                                                                                                                                                                       |
| **Distributed Bus** (cross-service messaging)                                                                                                                                                                                        | Yes                                                                                                                                                                                                                     |
| **Multi-tenancy** — tenant-isolated event streams, projections, channels                                                                                                                                                             | Yes                                                                                                                                                                                                                     |
| **Tenant-routed message channels**                                                                                                                                                                                                   | Yes — messages auto-routed to per-tenant queues                                                                                                                                                                         |
| **Priority routing** (VIP/standard queues by runtime context)                                                                                                                                                                        | Yes                                                                                                                                                                                                                     |
| **End-to-end PII encryption** (event store + broker messages + log output)                                                                                                                                                           | Yes — `#[Sensitive]` attribute; channel-level `ChannelProtectionConfiguration` for payload + named headers                                                                                                              |
| **Automatic correlation / causation propagation**                                                                                                                                                                                    | Yes — correlation ID and parent-message ID travel from command to every emitted event with no middleware; full OpenTelemetry spans stitch themselves                                                                    |
| **OpenTelemetry tracing**                                                                                                                                                                                                            | Yes — sync and async hops                                                                                                                                                                                               |
| **Interceptors** (cross-cutting concerns via attributes)                                                                                                                                                                             | Yes                                                                                                                                                                                                                     |
| **Framework portability** (Laravel ↔ Symfony ↔ PSR-11)                                                                                                                                                                               | Yes                                                                                                                                                                                                                     |
| **In-process async testing** (EcotoneLite)                                                                                                                                                                                           | Yes                                                                                                                                                                                                                     |

[Read more: Why Ecotone?](why-ecotone.md)

***

## Where Ecotone fits in your stack

**Your framework stays. Your ORM stays. Your queues and transports stay. Your deployment stays.** Ecotone is a Composer package that adds architecture on top of what you already run — business logic as POPOs, messaging topology as attributes, and your existing Laravel, Symfony, or PSR-11 container underneath.

***

## AI-ready by design

Ecotone's declarative, attribute-based architecture is inherently friendly to AI code generators. When your AI assistant works with Ecotone code, two things happen:

**Less context needed, less code generated.** A command handler with `#[CommandHandler]` and `#[Asynchronous('orders')]` tells the full story in two attributes — no bus configuration files, no handler registration, no retry setup to feed into the AI's context window.

**AI that knows Ecotone.** Your AI assistant can work with Ecotone out of the box:

* [**Agentic Skills**](other/ai-integration.md) — Ready-to-use skills that teach any coding agent how to correctly write handlers, aggregates, sagas, projections, tests, and more.
* [**MCP Server**](other/ai-integration.md) — Direct access to Ecotone documentation for any AI assistant that supports Model Context Protocol — Claude Code, Cursor, Windsurf, GitHub Copilot, and others.
* [**llms.txt**](other/ai-integration.md) — AI-optimized documentation files that give any LLM instant context about Ecotone's API and patterns.

Declarative configuration that any coding agent can follow and reproduce. Testing support that lets it verify even the most advanced flows. Less guessing, no hallucinating — just confident iteration.

***

## Start with your framework

**Laravel** — Laravel's queue runs jobs, not business processes. Stop stitching Spatie + Durable Workflow + `Bus::chain` + DIY outbox. Ecotone replaces the patchwork with one attribute-driven toolkit: aggregates with auto-published events, piped workflows, sagas, snapshots, transactional outbox — testable in-process, running on the queues you already have.\
`composer require ecotone/laravel`\
→ [Laravel Quick Start](quick-start-php-ddd-cqrs-event-sourcing/laravel-ddd-cqrs-demo-application.md) · [Laravel Module docs](modules/laravel/)

**Symfony** — Symfony Messenger handles dispatch. For aggregates, sagas, or event sourcing, the usual path is bolting on a separate ES library, rolling your own outbox, and adding a saga or workflow layer on top. Ecotone replaces the patchwork with one attribute-driven toolkit: aggregates, sagas, event sourcing, piped workflows, transactional outbox, dedup, and per-handler failure isolation. Pure POPOs, Bundle auto-config, your Messenger transports preserved.\
`composer require ecotone/symfony-bundle`\
→ [Symfony Quick Start](quick-start-php-ddd-cqrs-event-sourcing/symfony-ddd-cqrs-demo-application/) · [Symfony Module docs](modules/symfony/)

**Any PHP framework** — Ecotone Lite works with any PSR-11 compatible container.\
`composer require ecotone/lite-application`\
→ [Ecotone Lite docs](modules/ecotone-lite/)

***

## Proven patterns, proven runtime

* In continuous development since 2017 — nine years of production.
* Maintained by a core team of three (Dariusz Gafka, Jean de La Bédoyère, Piotr Zając) and an open-source contributor community.
* **No breaking changes across major versions.** Ecotone follows a stability commitment — your business code keeps working across releases, so upgrades are safe to apply. See the [changelog](https://github.com/ecotoneframework/ecotone-dev/blob/main/CHANGELOG.md).
* **Commercial support, SLA, consulting, and workshops** available for teams running Ecotone in production — [contact us](other/contact-workshops-and-support.md) to arrange a support agreement.
* Built on Enterprise Integration Patterns — the same pattern language behind Spring Integration, Axon Framework, NServiceBus, MassTransit, and Apache Camel.
* AI-ready: MCP server, ready-to-use coding-agent skills, `llms.txt` docs for any LLM.

***

## Evaluate it in one handler

You do not need to migrate anything to try Ecotone. You do not need architectural buy-in to evaluate it. Install the package, add `#[CommandHandler]` to one method, run your tests, and decide.

* If the pattern fits, add more attributes alongside your existing code — nothing in the rest of your codebase needs to change.
* If it doesn't fit for this project, remove the package before you've invested. The evaluation is intentionally reversible; the architectural commitment happens later, as your domain actually grows into the deeper features.

**Five-minute install. Familiar patterns from day one. No forced architectural migrations as your system grows.**

* [Install](install-php-service-bus.md) — Setup guide for any framework
* [Composing Building Blocks](composing-building-blocks.md) — Full walk-through of the composition model with Messenger / Laravel comparisons
* [Learn by example](quick-start-php-ddd-cqrs-event-sourcing/) — Send your first command in 5 minutes
* [Go through tutorial](tutorial-php-ddd-cqrs-event-sourcing/) — Build a complete messaging flow step by step
* [Workshops, Support, Consultancy](other/contact-workshops-and-support.md) — Hands-on training for your team

{% hint style="success" %}
Join [Ecotone's Community Channel](https://discord.gg/GwM2BSuXeg) — ask questions and share what you're building.
{% endhint %}
