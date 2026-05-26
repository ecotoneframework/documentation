---
description: >-
  The PHP workflow engine — durable, identifier-mapped, broker-agnostic
  workflows declared in plain PHP on the database and broker you already
  operate.
---

# The PHP Workflow Engine

## Durable, identifier-mapped, broker-agnostic workflows in PHP

Ecotone is the workflow engine for PHP. It delivers durable execution, identifier-mapped sagas, declarative orchestrators, chained handlers, timeouts, outbox, deduplication, retry/DLQ, per-handler failure isolation, distributed routing, and multi-tenant channels in **one attribute-driven model** on Laravel, Symfony, or Ecotone Lite.

Workflows are plain PHP classes with attributes. State lives in your own database. Execution runs on the broker you already operate.

```bash
composer require ecotone/laravel    # or ecotone/symfony-bundle
```

## Durable execution on your own database

Workflows survive crashes, restarts, and rolling deploys because state and dispatch live in infrastructure you already operate — no external runtime to deploy, no engine-shaped event history to maintain.

**Step 1 — State persisted per identifier.** A `#[Saga]` records its state in your own database, keyed by the correlation identifier the events carry.

**Step 2 — State + dispatch commit together.** `CombinedMessageChannel` writes the business state change and the outgoing message into one DBAL transaction — no dual-write window.

**Step 3 — Crash recovery is broker redelivery.** Worker dies mid-step? The broker redelivers the message; the saga reloads from the DB by identifier; built-in deduplication tolerates the duplicate; the work resumes.

### What this gives you

- Workers can be killed mid-flow; no work is lost — the broker holds the message until the next consumer picks it up.
- Deploys are no different from any rolling deploy of your app — no special workflow-version migration, no engine state to coordinate.
- The data stays in your own database — your backups, your retention, your access controls, your existing operational tooling.
- New subscribers and projections can be added against the same event stream because it lives in your DB and on your broker — directly queryable, directly extensible.

## Three workflow shapes, one model

### `#[Saga]` — stateful process manager

For processes that react to events arriving over time. State is persisted per `#[Identifier]`; on each event arrival, Ecotone reloads the saga from the database. Event-to-saga binding via payload field, header, or expression (`identifierMapping`). `#[Delayed]` handles saga timeouts without cron.

```php
#[Saga]
final class OrderFulfillment
{
    public function __construct(
        public string $orderId,
        public string $status,
    ) {}

    #[EventHandler(identifierMapping: ['orderId' => 'orderId'])]
    public static function start(OrderPlaced $event): self
    {
        return new self($event->orderId, 'awaiting-payment');
    }

    #[EventHandler(identifierMapping: ['orderId' => 'orderId'])]
    public function onPaymentCompleted(PaymentCompleted $event, CommandBus $commandBus): void
    {
        $this->status = 'awaiting-shipment';
        $commandBus->send(new ShipOrder($this->orderId));
    }

    #[Delayed(new TimeSpan(hours: 24))]
    #[EventHandler(identifierMapping: ['orderId' => 'orderId'])]
    public function onPaymentTimeout(OrderPlaced $event, CommandBus $commandBus): void
    {
        if ($this->status === 'awaiting-payment') {
            $commandBus->send(new CancelOrder($this->orderId));
        }
    }
}
```

### `#[Orchestrator]` — declarative routing-slip workflow

For processes whose step list is visible up front and where each step is reusable. The orchestrator returns the list of channel names for the next steps — including dynamic step lists computed from input data.

```php
final class CheckoutOrchestrator
{
    #[Orchestrator(inputChannelName: 'checkout')]
    public function plan(CheckoutRequest $request): array
    {
        $steps = ['validate-cart', 'reserve-inventory', 'charge-payment'];
        if ($request->isGift) {
            $steps[] = 'attach-gift-message';
        }
        $steps[] = 'send-confirmation';
        return $steps;
    }
}
```

Each channel name above is implemented by an independently testable `#[InternalHandler]`.

### Chained `#[InternalHandler]` — stateless durable flow

For workflows where the message itself carries the state. `outputChannelName` advances the message to the next handler. When the async channel runs through an outbox, each step is committed atomically with its business write — if a worker crashes, the broker redelivers and work resumes on the next consumer.

```php
final class IngestPipeline
{
    #[InternalHandler(inputChannelName: 'ingest.start', outputChannelName: 'ingest.validate')]
    public function fetch(IngestRequest $request): RawDocument { /* ... */ }

    #[InternalHandler(inputChannelName: 'ingest.validate', outputChannelName: 'ingest.store')]
    public function validate(RawDocument $doc): ValidatedDocument { /* ... */ }

    #[InternalHandler(inputChannelName: 'ingest.store')]
    public function store(ValidatedDocument $doc): void { /* ... */ }
}
```

## Five properties Ecotone delivers as primitives

### Crash survival

Worker killed mid-step? The broker redelivers; the saga reloads from its identifier; the handler runs again. Built-in deduplication tolerates the duplicate. State + dispatch commit together via `CombinedMessageChannel`, so there is no dual-write window to leak inconsistent state.

### Time-shifted steps

`#[Delayed]` resumes a handler after a `TimeSpan`, an exact `DateTime`, or an expression — no cron, no separate timer service. The broker's underlying delayed-message primitive does the waiting; the attribute model hides the broker specifics from your code.

### Per-step retry isolation

Each subscriber receives its own copy of the message. A failing handler retries on its own envelope; sibling handlers are unaffected. Retry strategy is per-handler, not per-transport — the failure of one step never replays a side effect on another.

### Identifier-mapped correlation

`#[Saga]` binds events to instances by payload field, header, or expression via `identifierMapping`. State is loaded by identifier on every event arrival; the saga remembers where it is across hours, days, or months of events.

### Cross-service routing

`#[Distributed]` handlers and the Distributed Bus extend the same workflow primitives across bounded contexts. Service Map carries the topology; multi-broker single-topology is first-class. The retry, DLQ, idempotency, and outbox semantics that protect in-process steps apply uniformly across service boundaries.

## The shape of every PHP workflow scenario

| Scenario | Ecotone shape |
|---|---|
| Multi-step order, payment, or fulfillment process | `#[Orchestrator]` declares the step list; each step is an `#[InternalHandler]`; the outbox commits state + dispatch atomically. |
| Long-running onboarding with timeouts and human waits | `#[Saga]` holds state across days or weeks; `#[Delayed]` handlers fire after a `TimeSpan` or `DateTime`; one identifier carries human-approval, timeout, and resume events. |
| Cross-service business processes spanning bounded contexts | `#[Distributed]` handlers exchange commands and events over your broker; Service Map declares the topology; adding a service is a config change. |
| High-throughput event-driven workflows with strict isolation | Per-handler failure isolation: each subscriber consumes its own copy; `CombinedMessageChannel` separates outbox storage from execution; consumers scale horizontally. |
| Multi-tenant workflows with isolated per-tenant channels | Dynamic channels route per-tenant via headers at runtime; one deployment serves every tenant; per-channel retry, priority, DLQ. |

## Workflow primitives

| Attribute | Purpose |
|---|---|
| `#[Saga]` | Stateful process manager. `#[Identifier]` maps incoming events to the right instance. State persisted in your DB; reloaded per event. |
| `#[Orchestrator]` | Routing-slip workflow. Returns the channel list for the next steps — including dynamic lists computed from input data. |
| `#[InternalHandler]` chains | Stateless durable flow. The message carries the state; `outputChannelName` advances it. Combined with the outbox, each step commits atomically. |
| `#[Delayed]` | Saga timeouts as attribute — `TimeSpan`, exact `DateTime`, or expression. The broker's delayed-message primitive handles the wait. |
| `CombinedMessageChannel` | One DBAL transaction commits business state and the outgoing message together. One poller drains the database; the broker carries execution. |
| `#[Distributed]` | Cross-service commands and events on the brokers you already operate. Service Map carries the topology. |
| Dynamic channels | Per-tenant message routing at runtime via headers. Multi-tenant workflows in one deployment. |
| `#[Deduplicated]` | Gateway-level deduplication absorbs redelivered messages, double-clicks, and webhook retries. |
| `#[Priority]`, `#[TimeToLive]` | Uniform attribute model across RabbitMQ, Kafka, SQS, Redis, DBAL, Messenger transports, and Laravel Queue channels. |

## In production

Workflows running where failure is regulated, expensive, or public:

- Payment gateways where a retried handler must never double-charge a customer.
- Credit card systems where transaction loss is catastrophic and the outbox is non-negotiable.
- Certification authorities whose event log is the audit log of record.
- E-commerce platforms orchestrating order, payment, and fulfillment sagas.
- Public transportation subscription systems coordinating nationwide transit subscriptions with Kafka integration to Java services.
- Two-sided marketplaces coordinating customer orders, provider subscriptions, and B2B partnerships.

## Continue

- [Durable Execution](durable-execution.md) — the durable-execution model in depth
- [Complex Business Processes](complex-business-processes.md) — saga and orchestrator patterns at length
- [Orchestration Layer](orchestration-layer.md) — process, service, and message-routing orchestration on one model
- [Microservice Communication](microservice-communication.md) — Distributed Bus and Service Map
