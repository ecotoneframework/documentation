---
description: Domain-Driven Design in PHP — Aggregates, Sagas as process managers, Bounded Contexts via Distributed Bus, Domain Events as first-class citizens, on Laravel and Symfony
---

# Domain-Driven Design in PHP

For a PHP team building DDD-shaped software, Ecotone's vocabulary *is* DDD vocabulary — Aggregate, Identifier, Repository, Domain Event, Saga (as process manager), Bounded Context — and each pattern is a declarative PHP attribute, not a base class to extend or a contract to implement.

## The Problem You Recognize

You've read Evans, you've read Vernon, you understand strategic and tactical DDD. The team has done the modelling work and identified bounded contexts. Now you're trying to express that model in PHP — and the tooling pushes back.

- Aggregates end up extending a base class from some DDD library that brings in its own command bus, its own event publication mechanism, and its own opinions about repositories.
- Domain events are dispatched through a *third* library's event dispatcher, which has its own subscription model that doesn't compose with the application's other event handlers.
- Sagas are home-grown state machines because the framework doesn't have a saga concept at all.
- Bounded contexts are aspirational — there's no operational boundary between them; one application database, one shared model, "bounded" only in the wiki.
- The PHP DDD libraries you've evaluated turned out to be effectively dormant (Prooph's `event-sourcing` and `service-bus` packages untouched since 2021; project site frozen at 2019), stable-but-not-evolving (Broadway's last functional release was May 2023; subsequent commits are CI and dependency bumps), or focused on one narrow slice (just ES, just aggregates, just buses).

You want a single, maintained framework where the DDD vocabulary is also the framework's vocabulary.

## What the Industry Calls It

**Domain-Driven Design** — model the business domain as the central concern of the software; use the *ubiquitous language* of domain experts in the code; segment the system into *bounded contexts* with explicit translations between them; encapsulate invariants inside *aggregates*; reflect state changes as *domain events*; coordinate long-running processes via *process managers* (often called sagas). Pattern catalog from Evans (2003), refined by Vernon (2013).

In PHP, faithful tactical DDD has historically meant assembling the building blocks from disparate libraries — and the libraries don't share retry policies, transaction boundaries, identity mappings, or event publication. Ecotone provides the full set on one model.

## How Ecotone Solves It

### Aggregates — invariants, command handlers, events

```php
#[Aggregate]
final class Customer
{
    #[Identifier] private string $customerId;
    private CustomerStatus $status = CustomerStatus::Active;
    private array $domainEvents = [];

    #[CommandHandler]
    public static function register(RegisterCustomer $command): self
    {
        $self = new self();
        $self->customerId = $command->customerId;
        $self->domainEvents[] = new CustomerRegistered($command->customerId, $command->email);
        return $self;
    }

    #[CommandHandler]
    public function suspend(SuspendCustomer $command): void
    {
        if ($this->status === CustomerStatus::Suspended) {
            return;
        }
        $this->status = CustomerStatus::Suspended;
        $this->domainEvents[] = new CustomerWasSuspended($this->customerId, $command->reason);
    }
}
```

Plain final class. `#[Aggregate]` marks the type, `#[Identifier]` declares the identity, command handlers live on the aggregate (`#[CommandHandler]` on a static factory for creation, on an instance method for state-changing operations). No base class, no interface. Domain events are plain PHP classes the aggregate records.

For event-sourced aggregates, `#[EventSourcingAggregate]` with `#[EventSourcingHandler]` event-application methods — see the [Event Sourcing](event-sourcing.md) landing for the full story.

### Repositories — abstraction without imposition

Repositories use the inbuilt DBAL or Eloquent repository, or implement the Repository contract yourself for any persistence model. The aggregate stays free of persistence concerns; the framework wires the repository in.

### Sagas as process managers

```php
#[Saga]
final class CustomerOnboarding
{
    #[Identifier] private string $customerId;
    private bool $emailVerified = false;
    private bool $kycPassed = false;

    #[EventHandler]
    public static function start(CustomerRegistered $event, CommandBus $bus): self
    {
        $bus->send(new RequestEmailVerification($event->customerId, $event->email));
        $bus->send(new InitiateKyc($event->customerId));
        return new self($event->customerId);
    }

    #[EventHandler]
    public function onEmailVerified(EmailVerified $event): void
    {
        $this->emailVerified = true;
        $this->completeIfReady();
    }

    #[EventHandler]
    public function onKycPassed(KycPassed $event): void
    {
        $this->kycPassed = true;
        $this->completeIfReady();
    }
}
```

The saga is a process manager in Vernon's sense — it remembers state across events arriving over time, decides what to do next, and dispatches commands to keep the process moving. State persisted per `#[Identifier]`; on each event arrival, Ecotone reloads the saga from the database. Event-to-saga binding via payload field, header, or expression (`identifierMapping`).

### Bounded Contexts — operational, not aspirational

The Distributed Bus and Service Map (Enterprise) move commands and events between PHP services over the brokers you operate. Each bounded context runs as its own deployable with its own database, communicating with other contexts via well-defined domain events on the bus. The translation between contexts becomes explicit: events emitted by Context A are subscribed to by translators in Context B (using `#[Distributed]` handlers on each side). Strategic DDD made operational.

### Domain Events as first-class citizens

Domain events are plain PHP classes — no `DomainEventInterface`, no `recordThat()` mixin required by the framework. The aggregate records events on itself; Ecotone publishes them onto the event bus when the aggregate is persisted. Subscribers (other aggregates, sagas, projections, integration handlers) bind via `#[EventHandler]` with optional `identifierMapping` or routing constraints.

### Value Objects and conversion

Value objects pass through the JMS Converter pipeline cleanly — serialize through type converters; deserialize on the way in. The same `#[Sensitive]` attribute that encrypts fields in the event store encrypts them in messages on the broker, so PII boundaries respect the domain's classification.

## How It Compares

| Dimension | Broadway | Prooph | Spatie laravel-event-sourcing | Assemble it yourself | Ecotone |
| --- | --- | --- | --- | --- | --- |
| Maintenance status | Stable since May 2023 — recent commits are CI / dependency bumps | Core `event-sourcing` and `service-bus` packages untouched since 2021; site frozen at 2019 | Active | Yours forever | Active, in production since 2019 |
| Framework support | Symfony-leaning (broadway/broadway-bundle) | Framework-agnostic | Laravel only | You pick | Laravel + Symfony + Ecotone Lite |
| Aggregate model | Yes | Yes | Yes | Hand-roll | `#[Aggregate]` / `#[EventSourcingAggregate]` |
| Saga as process manager | Limited | Yes (Prooph Service Bus) | Reactors, not full sagas | Hand-roll a state machine | `#[Saga]` with `#[Identifier]` mapping |
| Bounded-context distribution | Out of scope | Manual | Out of scope | Six libraries | Distributed Bus + Service Map |
| Repository abstraction | Yes | Yes | Eloquent-bound | You pick | DBAL / Eloquent / custom |
| Shared message-driven middleware (retry, DLQ, dedup, OTel, PII) | No | Partial | No | Per-library | One model across every building block |
| Domain Event publication | Library-specific | Library-specific | Library-specific | Hand-roll | Plain PHP events on the event bus |

PHP's DDD library landscape in 2026 is sparse: Broadway has been on dependency-bump-only maintenance since its May 2023 functional release, Prooph's `event-sourcing` and `service-bus` packages haven't received a commit since 2021, and Spatie's library scopes to Laravel-only event sourcing. Ecotone covers the tactical DDD vocabulary as one declarative model on Laravel, Symfony, or Ecotone Lite.

## Next Steps

- [CQRS Introduction](../modelling/command-handling/) — message buses, command handlers, query handlers, event handlers
- [Aggregate Introduction](../modelling/command-handling/state-stored-aggregate/) — state-stored aggregates
- [Event Sourcing](event-sourcing.md) — event-sourced aggregates and projections
- [Sagas](../modelling/business-workflows/sagas.md) — process managers
- [Identifier Mapping](../modelling/command-handling/identifier-mapping.md) — binding events to aggregates and sagas
- [Repositories](../modelling/command-handling/repository/) — persistence abstraction
- [Microservice Communication](microservice-communication.md) — Bounded Contexts on the wire
- [PHP for Enterprise Architecture](php-for-enterprise-architecture.md) — the architecture-layer framing

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Orchestrators](../modelling/business-workflows/orchestrators.md) for declarative multi-step workflows, [Distributed Bus with Service Map](../modelling/microservices-php/distributed-bus/distributed-bus-with-service-map/) for bounded-context distribution across multiple brokers, [Streaming Projections](../modelling/event-sourcing/setting-up-projections/scaling-and-advanced.md) over Kafka, and [Advanced Event Sourcing Handlers](../modelling/event-sourcing/event-sourcing-introduction/) for context-aware aggregate reconstruction.
{% endhint %}
