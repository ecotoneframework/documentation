---
description: Durable execution in PHP — sagas, workflows, outbox, and retries on the database and broker you already run, with no separate cluster
---

# Durable Execution

## The Problem You Recognize

A multi-step business process — order fulfillment, subscription provisioning, payment + payout — spans minutes to days. You need it to **survive crashes**: if the worker dies mid-step, the process picks up where it left off. If a downstream system is briefly unavailable, the step retries. If business state changes, every consumer sees a consistent timeline.

You've looked at Temporal. It's a real answer — but the answer comes with:

- **A separate runtime to operate.** Temporal runs as its own service (Frontend / History / Matching / internal Worker components) with a dedicated workflow database and a separate visibility store — two stateful systems alongside your application's own database. That's a design decision, not a version-specific quirk.
- **Debugging happens on someone else's runtime.** Because workflow code executes inside the Temporal worker runtime (RoadRunner + ext-grpc on PHP), the workflow doesn't run where your application runs — different process, different debugger surface, different mental model.
- **Workflow history in an engine-specific format** — switching off the platform is a rewrite, and projections or new subscribers can't be built from the workflow's event history later.

For most PHP teams already running PostgreSQL or MySQL and RabbitMQ / Kafka / SQS / Redis, this is a new runtime, a new programming model, and a new lock-in surface — to solve a problem the existing stack can already solve.

## What the Industry Calls It

**Durable Execution** — the umbrella term for multi-step processes that resume correctly across failures. It's a composition of older patterns, not a new one:

- **Sagas** — stateful long-running coordinators that remember where they are across events arriving over time.
- **Workflows / Orchestrators** — declarative step sequences that hand a message from one handler to the next.
- **Outbox** — atomic publication of a message together with the database change that produced it.
- **Retry + Dead Letter** — automatic recovery for transient failures, with a quarantine for the rest.
- **Event Sourcing** — every state change is a durable event, so rebuilding state on restart is replaying a list of events you already own.

When these compose, processes are durable. The infrastructure underneath them is whatever the application already runs.

## How Ecotone Solves It

Ecotone delivers durable execution as a **Composer package on top of your existing database and broker**. No separate runtime, no constrained workflow DSL — workflows are plain PHP classes with attributes, and the durability primitives live in the database you already have.

### Outbox in one transaction, execution on the broker you already run

The outbox pattern is one configuration line. `CombinedMessageChannel` writes the message into the database in the **same transaction** as the business state change, then dispatches the actual handler execution onto your broker (RabbitMQ / Kafka / SQS / Redis / …). One pollers handles outbox draining; many consumers handle broker-side execution — scale workers on the broker, not the database.

```php
#[ServiceContext]
public function outboxToSqs(): CombinedMessageChannel
{
    return CombinedMessageChannel::create(
        'outbox_sqs',
        ['database_channel', 'amazon_sqs_channel'],
    );
}

#[Asynchronous(['outbox_sqs'])]
#[EventHandler]
public function notifyAboutNewOrder(OrderWasPlaced $event): void
{
    // Message committed in the same DBAL transaction as the business write.
    // Execution dispatched to SQS where workers scale horizontally.
}
```

### Sagas — stateful processes in plain PHP

A Saga is a plain class with `#[Saga]`, an `#[Identifier]`, and event handlers. State is persisted per identifier; on the next event arrival, Ecotone reloads the saga from the database. No replay constraints, no `Date::now()` restrictions, no versioning branches.

```php
#[Saga]
final class OrderFulfillment
{
    #[Identifier] private string $orderId;
    private string $status = 'placed';

    #[EventHandler]
    public static function start(OrderWasPlaced $event): self
    {
        return new self($event->orderId);
    }

    #[EventHandler]
    public function onPaymentReceived(PaymentReceived $event, CommandBus $bus): void
    {
        $this->status = 'paid';
        $bus->send(new ShipOrder($this->orderId));
    }
}
```

### Saga timeouts — `#[Delayed]` on a saga event handler

Long-running processes need to time out: verify a phone number within 24 hours or block the user; complete checkout within 15 minutes or release the cart. With Ecotone, a timeout is one extra event handler on the saga, delayed by an attribute. No cron, no scheduled job, no separate timer service.

```php
#[Saga]
final class VerificationProcess
{
    #[Identifier] private string $userId;
    private bool $emailVerified = false;
    private bool $phoneVerified = false;

    #[EventHandler]
    public static function start(UserWasRegistered $event): self
    {
        // also publishes VerificationProcessStarted which the timeout below listens to
    }

    #[Delayed(new TimeSpan(hours: 24))]
    #[Asynchronous('async')]
    #[EventHandler(endpointId: 'verification.timeout')]
    public function timeout(VerificationProcessStarted $event, CommandBus $bus): void
    {
        if ($this->emailVerified && $this->phoneVerified) {
            return;
        }
        $bus->sendWithRouting('user.block', metadata: ['aggregate.id' => $this->userId]);
    }
}
```

`#[Delayed]` accepts a `TimeSpan`, an exact `\DateTimeImmutable`, or an expression — `#[Delayed(expression: 'payload.dueDate')]` will fire when the dueDate field on the event arrives. The delay is enforced per-handler on the async channel, so the *same event* can trigger an immediate handler and a 24-hour-delayed handler without colliding.

### Stateless workflows — durable without a persistent saga

Not every durable flow needs a stateful coordinator. Many multi-step processes are just *chained handlers* — payment → ship → notify — where nothing needs to be remembered between steps; the message itself carries the state. With Ecotone, you connect handlers through `outputChannelName`, and run those channels asynchronously. The architecture stays simple and fast: no saga record, no aggregate hydration, no extra database round trips per step.

```php
#[CommandHandler(routingKey: 'order.place', outputChannelName: 'order.verify_payment')]
public function placeOrder(PlaceOrder $command): OrderData { /* ... */ }

#[Asynchronous('async')]
#[InternalHandler(inputChannelName: 'order.verify_payment', outputChannelName: 'order.ship')]
public function verifyPayment(OrderData $order): OrderData { /* ... */ }

#[Asynchronous('async')]
#[InternalHandler(inputChannelName: 'order.ship', outputChannelName: 'order.notify')]
public function ship(OrderData $order): OrderData { /* ... */ }

#[Asynchronous('async')]
#[InternalHandler(inputChannelName: 'order.notify')]
public function notifyCustomer(OrderData $order): void { /* ... */ }
```

**Durability comes from the channel, not from saga state.** When the `async` channel runs through an outbox (`CombinedMessageChannel` writing to the database), each step is committed atomically with its business write before the message advances to the next handler. If a worker crashes mid-step, the broker redelivers the message; the work resumes on the next consumer. Ecotone delivers the same recovery semantics another way: redelivery of the message, idempotency through built-in deduplication, and the outbox table living in your own database — so the durable state and the business state commit together or roll back together.

For declarative workflows where the *step list itself* lives in one place — including dynamic step lists chosen from input data — Ecotone Enterprise provides [Orchestrators](../modelling/business-workflows/orchestrators.md).

### Retry, error channels, dead letter — at the channel, not per handler

Configure recovery policy once at the channel level. Every handler on that channel inherits retries with exponential backoff, an error channel, and a DBAL-backed dead letter queue. No per-handler boilerplate.

```php
ErrorHandlerConfiguration::createWithDeadLetterChannel(
    'error_channel',
    RetryTemplateBuilder::exponentialBackoff(1000, 2)->maxRetryAttempts(3),
    'dbal_dead_letter',
)
```

### Event Sourcing — durability you can query

When the business process *is* its history (audit trails, regulated domains, reconstructible read models), Event Sourcing gives durable execution as a side effect: every step is a recorded event in your own database, in your own schema, queryable by the rest of your application — not locked inside a workflow engine's internal log.

### Test any flow in isolation with EcotoneLite

The same programming model runs in production *and* in your test suite. `EcotoneLite::bootstrapFlowTesting` boots a real Ecotone application in-process — buses, sagas, projections, async channels, outbox — and runs flows synchronously inside the test. No queue infrastructure, no separate worker process, no flakiness. Just call the bus, run the consumer, assert the result.

```php
$test = EcotoneLite::bootstrapFlowTesting(
    [OrderService::class, NotificationService::class, OrderFulfillment::class],
    [new OrderService(), new NotificationService()],
    enableAsynchronousProcessing: [
        SimpleMessageChannelBuilder::create('async'),
    ],
);

$test->sendCommandWithRoutingKey('order.place', new PlaceOrder('order-1', amount: 4_200));
$test->run('async'); // drain the async channel

self::assertSame('paid', $test->sendQueryWithRouting('order.status', metadata: ['aggregate.id' => 'order-1']));
```

What this buys you:

- **Same model sync and async.** A test runs through the outbox, the broker channel, the saga, the projection — the same code path production hits. There's no separate "test mode" that diverges from runtime behaviour.
- **Local equals production, and the runtime is yours.** Drop a `var_dump`, set a breakpoint, step through the saga line by line — everything runs in one PHP process, in your existing debugger. There's no separate workflow runtime sitting between you and your code; the saga executes where your application executes.
- **Time-travel a timeout in seconds.** Async channels can be driven manually (`->run('async')`) and clock-based delays can be advanced explicitly, so a 24-hour saga timeout becomes a one-line test, not an integration suite that waits.
- **Confidence that ships with the code.** You build it, you test it end-to-end against the real Ecotone runtime — so when an incident lands at 3 a.m., the production behaviour is the behaviour you've already debugged on your laptop.

## What the Code Actually Looks Like

The clearest difference between Temporal and Ecotone is what a workflow *looks like* when you write it. The same business process — placing an order, charging payment, shipping, notifying the customer — in both frameworks:

### Temporal PHP SDK — workflow + activity proxies

```php
#[WorkflowInterface]
class OrderFulfillmentWorkflow
{
    private $payments;
    private $shipping;
    private $notifications;
    private string $status = 'placed';

    public function __construct()
    {
        $options = ActivityOptions::new()
            ->withStartToCloseTimeout(CarbonInterval::seconds(30));
        $this->payments      = Workflow::newActivityStub(PaymentActivity::class, $options);
        $this->shipping      = Workflow::newActivityStub(ShippingActivity::class, $options);
        $this->notifications = Workflow::newActivityStub(NotificationActivity::class, $options);
    }

    #[WorkflowMethod]
    public function fulfill(string $orderId, int $amount): \Generator
    {
        // No Date::now() — must use Workflow::now() so replay stays deterministic.
        $paymentId = yield $this->payments->charge($orderId, $amount);
        $this->status = 'paid';

        // No sleep() — must use Workflow::timer() so the wait is recorded in history.
        yield Workflow::timer(CarbonInterval::seconds(2));

        $shipmentId = yield $this->shipping->ship($orderId);
        $this->status = 'shipped';
        yield $this->notifications->notifyCustomer($orderId, $shipmentId);
        return [$paymentId, $shipmentId];
    }

    #[QueryMethod]   public function status(): string { return $this->status; }
    #[SignalMethod]  public function cancel(): void   { /* ... */ }
}
```

Things to notice: the workflow method is a `Generator` that `yield`s through activity proxies; you can't any direct service inside it; every external call must go through an activity stub configured up front; signals, queries, and the workflow method are three separate APIs.

### Ecotone — plain PHP saga, no proxies, no DSL

```php
use Ecotone\Modelling\Attribute\Saga;
use Ecotone\Modelling\Attribute\Identifier;
use Ecotone\Modelling\Attribute\EventHandler;
use Ecotone\Modelling\Attribute\QueryHandler;
use Ecotone\Modelling\CommandBus;

#[Saga]
final class OrderFulfillment
{
    #[Identifier] private string $orderId;
    private string $status = 'placed';
    private ?string $paymentId = null;
    private ?string $shipmentId = null;

    private function __construct(string $orderId)
    {
        $this->orderId = $orderId;
    }

    #[EventHandler]
    public static function start(OrderWasPlaced $event, CommandBus $bus): self
    {
        $bus->send(new ChargeOrder($event->orderId, $event->amount));
        return new self($event->orderId);
    }

    #[EventHandler]
    public function onPaymentCompleted(PaymentCompleted $event, CommandBus $bus): void
    {
        $this->paymentId = $event->paymentId;
        $this->status = 'paid';
        $bus->send(new ShipOrder($this->orderId));
    }

    #[EventHandler]
    public function onShipped(OrderShipped $event, CommandBus $bus): void
    {
        $this->shipmentId = $event->shipmentId;
        $this->status = 'shipped';
        $bus->send(new NotifyCustomer($this->orderId, $event->shipmentId));
    }

    #[EventHandler]
    public function onNotified(CustomerNotified $event): void
    {
        $this->status = 'done';
    }

    #[QueryHandler('order.status')]
    public function status(): string
    {
        return $this->status;
    }
}
```

The handlers *are* the steps. No activity interfaces, no activity proxies, no generators, no `Workflow::now()`, no `Workflow::timer()` — each handler is an ordinary message handler that can call any service, use any clock, do any I/O. Persistence happens automatically per `#[Identifier]`. Retries and dead-lettering come from the channel the handler runs on. Time delays use `#[Delayed]`, which looks identical to any other message attribute. Testing runs in-process with `EcotoneLite::bootstrapFlowTesting`.

> Aggregate itself can be combined with Doctrine ORM, Eloquent, so it does use the programming model you know. Saga can also be Event Sourced if you want to persist all the transition changes, or even build Projections around it.

**Both stacks have invisible scaffolding.** What you see above isn't all the code either side needs.

- *Temporal* also requires: an Activity implementation class for each `#[ActivityInterface]`, a Worker registration file binding workflow + activities to a Task Queue, a RoadRunner config (`.rr.yaml`), `ext-grpc` installed everywhere, and the running Temporal Server (Frontend + History + Matching + Worker) with its workflow database and visibility store.
- *Ecotone* also requires: a channel registration (one `#[ServiceContext]` method like the `CombinedMessageChannel` shown earlier), retry/DLQ configuration (one `ErrorHandlerConfiguration`), and the consumer process (`php bin/console ecotone:run` or the Laravel artisan equivalent).

The difference isn't scaffolding count — it's what the programming model lets you write *inside* the workflow file. Temporal demands a constrained DSL; Ecotone is plain PHP.

## How It Compares to Temporal

| Dimension | Temporal (PHP SDK) | Ecotone |
| --- | --- | --- |
| Infra you must run | Temporal Server (Frontend / History / Matching / internal Worker) with its own workflow database and a separate visibility store — alongside your application's database | Composer package on the database and broker you already run |
| How your existing broker fits in | Reached *through Activities*; the workflow runtime is Temporal's own routing fabric, not RabbitMQ / Kafka / SQS | First-class as the workflow runtime itself — RabbitMQ, Kafka, Redis, SQS, Enqueue, Symfony Messenger, Laravel Queue |
| Worker runtime | RoadRunner + ext-grpc — workflows execute inside Temporal's worker process | `php bin/console ecotone:run` / `php artisan ecotone:run` — handlers execute inside your application process |
| Workflow code | Replay-deterministic — no Date::now, no random, no direct I/O outside Activities | Plain PHP classes with attributes |
| Versioning long-running flows | `getVersion()` / `patched()` branches in workflow code that survive until the last in-flight execution finishes | Plain code change with schema discipline — add fields with defaults; drain in-flight sagas before breaking changes to persisted state |
| Durability primitive | Per-workflow Event History (engine-specific format) | Your own database rows / event store you can query |
| Outbox (atomic business write + message) | Design idempotent Activities + manual DB-write-then-publish in a dedicated Activity | Declarative `CombinedMessageChannel` in one DBAL transaction |
| Multi-tenancy | One namespace per tenant; cluster topology grows with tenant count | Header-routed channels in one deployment |
| Migration cost off-platform | Rewrite — engine-specific Event History; in-flight workflows can't be exported | Handlers and channel config are Ecotone-shaped, but saga state and event stream stay in your own schema, queryable from any tool during a transition |

## Already Running Temporal? Ecotone Composes With It

Durable workflows are one capability inside a wider system. Temporal handles them with replay-based state restore and a forensic UI over the workflow's history. Those are genuine strengths. If your team has already invested in Temporal, you don't have to choose.

### What Ecotone handles around Temporal

Ecotone is designed to fit *around* a workflow engine and handle everything Temporal doesn't:

- **CQRS and message buses.** Command, Event, and Query buses on Laravel or Symfony, registered through PHP attributes — used inside HTTP controllers, console commands, scheduled jobs, *and* inside Temporal Activities to dispatch work into the rest of the application.
- **Domain event publication.** Temporal's Event History is internal to the workflow. Ecotone publishes domain events from your aggregates onto your own broker (RabbitMQ / Kafka / SQS / Redis) so other services and other read models can subscribe — including projections you only realise you need months after the workflow shipped.
- **Read models and projections.** Build CQRS read sides — partitioned, streaming, replayable, blue-green deployable — from the events your application emits. Temporal's history is the wrong shape for this; your own event stream is the right shape.
- **Outbox.** When a Temporal Activity needs to write to the database *and* publish an external event atomically, Ecotone's `CombinedMessageChannel` gives you that in one DBAL transaction. The Activity stays short and idempotent; Ecotone handles the dual-write problem.
- **Sagas that aren't workflows.** Plenty of stateful coordination doesn't need full workflow durability — webhook deduplication windows, subscription grace periods, multi-event correlations. Ecotone Sagas handle these on the same database the application already uses.
- **Distributed messaging between services.** Ecotone's Distributed Bus and Service Map move commands and events between PHP services over the brokers you operate. Temporal's cross-cluster Nexus is its own thing; for the everyday between-services traffic of a PHP estate, Ecotone is the lighter fit.
- **Multi-tenant routing.** Header-routed channels in one deployment, instead of a namespace-per-tenant model that compounds with Temporal's cluster sizing.
- **Resilience on everything that *isn't* a workflow.** Retry, error channels, DBAL dead-letter, idempotency, per-handler failure isolation — applied uniformly to every async event handler in the system, workflow-bound or not.

A reasonable composition: Temporal runs the workflows that genuinely need its replay semantics; Ecotone runs the rest of the message-driven system around them. Activities call into Ecotone's command bus to keep the workflow body small; aggregates and projections live in Ecotone's event store; the broker is shared.

For PHP teams that haven't yet committed to Temporal — already on Laravel or Symfony with a database and broker they own — Ecotone is usually enough on its own: sagas, orchestrators, outbox, retries, and event sourcing reach the same durability without adding a cluster.

## Next Steps

- [Outbox Pattern](../modelling/recovering-tracing-and-monitoring/resiliency/outbox-pattern.md) — atomic message + business write
- [Sagas](../modelling/command-handling/saga.md) — stateful long-running coordinators
- [Complex Business Processes](complex-business-processes.md) — workflows and sagas side by side
- [Retries](../modelling/recovering-tracing-and-monitoring/resiliency/retries.md) — retry policy at the channel level
- [Error Channel and Dead Letter](../modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/) — failure quarantine
- [Event Sourcing](../modelling/event-sourcing/) — durability as a recorded history you own

{% hint style="success" %}
**As You Scale:** Ecotone Enterprise adds [Orchestrators](../modelling/business-workflows/orchestrators.md) — declarative multi-step workflows with dynamic step lists; [Command Bus Instant Retries](../modelling/recovering-tracing-and-monitoring/resiliency/retries.md#customized-instant-retries) for synchronous commands; and [Gateway-Level Deduplication](../modelling/recovering-tracing-and-monitoring/resiliency/idempotent-consumer-deduplication.md#deduplication-with-command-bus) for exactly-once semantics across handlers.
{% endhint %}
