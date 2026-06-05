---
description: Why Ecotone — the PHP architecture layer that grows with your system, without rewrites
---

# Why Ecotone?

## Focus on business logic — Ecotone handles the wiring

Ecotone gives you a set of building blocks that speed up everyday development behind one clear, understandable model. Because every building block rests on the same messaging foundation, Ecotone becomes the wiring: you write the business logic, and Ecotone connects it in a decoupled way — whether steps run synchronously or asynchronously, within a single application or across many. Your core business logic stays the first-class citizen while the wiring and infrastructure are abstracted away — which is what keeps even the most complex applications simple to build.

It runs on the Laravel or Symfony you already use. You keep your ORM (Eloquent or Doctrine), your routing, your templates, your deployment. What Ecotone adds is the architecture layer — the discipline that keeps a system maintainable as complexity grows.

```bash
composer require ecotone/laravel    # or ecotone/symfony-bundle
```

## What developers get from the approach

- **Clean domain code.** Business logic lives in plain PHP classes — no base classes to extend, no marker interfaces, no framework types leaking into the domain. Attributes declare intent; Ecotone wires the behaviour.
- **One model instead of a library zoo.** Buses, aggregates, sagas, event sourcing, outbox, and distributed messaging sit on the same messaging foundation, so retry, deduplication, transactions, PII handling, and observability are configured once and apply everywhere — not reconciled across separate libraries with separate conventions.
- **No forced rewrites as you grow.** New capability is a new attribute next to the existing code, not a migration from one library to another. The same classes, the same codebase, the same team.
- **Patterns proven in other ecosystems.** The same Enterprise Integration Patterns that underpin mature Java and .NET architecture layers — brought to PHP as attribute-driven code on plain classes.
- **Testable by default.** The EcotoneLite harness exercises handlers, aggregates, sagas, projections, and whole flows — synchronous or asynchronous — without booting infrastructure.

## One Package That Grows With Your System

The core promise of Ecotone: **no forced architectural migrations as your domain grows**. Every other PHP choice commits to a ceiling on day one. Spatie laravel-event-sourcing has no sagas. EventSauce assembles everything around it. Patchlevel has no outbox or distributed bus. Symfony Messenger is dispatch-only; aggregates, ES, sagas, and outbox are all separate library decisions.

With Ecotone, you start with `#[CommandHandler]` on day one. You add `#[Asynchronous]` when you need async. You add `#[Saga]` when you need a stateful workflow. You add `#[EventSourcingAggregate]` when audit and replay become requirements. You add `#[DistributedBus]` when your system splits into services.

**The same codebase, the same classes, new attributes next to the old ones.** No library swap. No parallel stack. No "migration from library A to library B" week.

## Patterns Proven in Other Ecosystems — Now on PHP

Ecotone is built on [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/), the same foundation behind:

| Ecosystem | Pattern-driven architecture layer |
|-----------|-----------------------------------|
| **Java** | Spring Integration, Axon Framework |
| **.NET** | NServiceBus, MassTransit, Wolverine |
| **PHP** | **Ecotone** |

The patterns are decades-tested in banking, telecom, and logistics systems. Ecotone brings them to PHP as attribute-driven code on your existing Laravel or Symfony application — so your team writes POPOs, and Ecotone applies the patterns.

## What You Get, Mapped to Problems

Instead of learning pattern names first, start with the problem you're solving:

| Your problem | What the industry calls it | Ecotone feature |
|-------------|---------------------------|-----------------|
| Business logic is scattered across controllers and services | CQRS (Command Query Responsibility Segregation) | [Message Bus and CQRS](modelling/command-handling/) |
| You need a full audit trail and the ability to rebuild state | Event Sourcing | [Event Sourcing](modelling/event-sourcing/) |
| Complex multi-step processes are hard to follow and maintain | Sagas, Workflow Orchestration | [Business Workflows](modelling/business-workflows/) |
| Async processing is unreliable and hard to debug | Resilient Messaging | [Resiliency](modelling/recovering-tracing-and-monitoring/resiliency/) |
| Retried handlers re-run sibling handlers and produce duplicate side effects | Per-handler failure isolation | [Resiliency](modelling/recovering-tracing-and-monitoring/resiliency/) |
| Complex routing flows collapse into conditional dispatch code | EIP composable messaging (pipes, content-based routing, splitters) | [Business Workflows](modelling/business-workflows/) |
| Renaming a command class breaks in-flight messages | Endpoint-ID routing | [Message Bus and CQRS](modelling/command-handling/) |
| Services need to communicate reliably across boundaries | Distributed Messaging | [Distributed Bus](modelling/microservices-php/) |
| Webhooks arrive twice and double-charge customers | Deduplication, per-bus retry/DLQ policy | [Resiliency](modelling/recovering-tracing-and-monitoring/resiliency/) |
| Multiple tenants need isolated processing with their own queues and priorities | Multi-Tenant messaging | [Multi-Tenancy Support](messaging/multi-tenancy-support/) |

## Full Capability Matrix

Everything below is included out of the box — not assembled from separate libraries.

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

## How It Integrates

Ecotone plugs into your existing framework without requiring changes to your application structure.

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

## Trusted in Regulated Production

Ecotone runs in production at:

* **Payment gateways** — where retried handlers cannot double-charge, and the outbox must guarantee that every committed transaction produces exactly one downstream message.
* **Credit card systems** — where transaction loss is catastrophic, and every state change must be auditable and replayable.
* **Certification authorities** — whose entire business depends on reconstructible, tamper-evident audit trails; the event log is the audit log.
* **E-commerce platforms** — orchestrating order → payment → fulfillment → notification as declarative sagas with compensation.
* **Public transportation subscription systems** — managing nationwide transit subscriptions (create, renew, terminate) with distributed bus integration to Java and PHP services over Kafka.
* **Two-sided marketplaces** — coordinating customer orders, provider subscriptions, lead distribution, and B2B enterprise partnerships on one event-driven backbone.

The capabilities above are not hypothetical. They run in systems where failure is either regulated (audit, payments), expensive (double-charges, lost deliveries), or public (transit, marketplaces).

## Proven Patterns, Proven Runtime

* In continuous development since 2017 — nine years of production.
* Maintained by SimplyCodedSoftware and an open-source contributor community.
* **No breaking changes across major versions.** Ecotone follows a stability commitment — your business code keeps working across releases, so upgrades are safe to apply. See the [changelog](https://github.com/ecotoneframework/ecotone-dev/blob/main/CHANGELOG.md).
* **Commercial support, Support Agreements, consulting, and workshops** available for teams running Ecotone in production — [contact us](other/contact-workshops-and-support.md) to arrange a support agreement.

## Start Today, Grow Into Every Pattern

**Day one:** install the package, add `#[CommandHandler]` to one method, run your tests.

**Week one:** add `#[Asynchronous]` to move handlers off the request cycle.

**Month three:** add `#[Saga]` for your first multi-step business process; add the transactional outbox.

**Year two:** event-source the aggregates where audit matters; add distributed bus when your system splits into services.

Same classes. Same codebase. Same team. No forced migration between stages.

[Learn about Enterprise features](enterprise.md) for advanced multi-tenancy, distributed bus with service map, orchestrators, and production-grade Kafka.
