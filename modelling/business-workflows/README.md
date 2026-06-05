---
description: Three shapes of durable workflows in PHP — Stateless Workflows, Sagas, and Orchestrators. Pick the one that matches your process, on the database and broker you already run.
---

# Business Workflows

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

{% hint style="success" %}
**Building durable workflows in PHP?** Start with [Durable Execution in PHP](../../solutions/durable-execution.md) — the three workflow shapes Ecotone supports, the trade-offs between them, and the code side-by-side.
{% endhint %}

## Three Shapes of Durable Workflows

Every multi-step business process has one of three shapes. Pick the one that matches yours — **all three are durable**, all three survive crashes and deploys, all three run on the database and broker you already operate.

| Shape | Pick when | Durability mechanism | Where state lives |
| --- | --- | --- | --- |
| **Stateless Workflow** | Linear pipeline; the message carries everything between steps | Channel + outbox redelivery; idempotent handlers | The message in flight |
| **Saga** | State must persist across events arriving over time (hours, days, weeks) | Event-keyed persistence (DB), optional event sourcing for full replay | Your DB, rehydrated per `#[Identifier]` on the next event arrival |
| **Orchestrator** *(Enterprise)* | The step list itself is the workflow — possibly dynamic per input | Routing slip on the channel; per-step channel redelivery | The slip header on the message |

---

## Stateless Workflows — when the message is the state

Use when steps are tightly linear (verify → charge → ship → notify), nothing needs to be remembered between steps, and the message itself carries the state from one handler to the next. Handlers are chained through `outputChannelName`; durability comes from the channel (outbox + redelivery), not from a persisted record.

```php
#[CommandHandler(routingKey: 'order.place', outputChannelName: 'order.verify_payment')]
public function placeOrder(PlaceOrder $command): OrderData { /* ... */ }

#[Asynchronous('async')]
#[InternalHandler(inputChannelName: 'order.verify_payment', outputChannelName: 'order.ship')]
public function verifyPayment(OrderData $order): OrderData { /* ... */ }

#[Asynchronous('async')]
#[InternalHandler(inputChannelName: 'order.ship', outputChannelName: 'order.notify')]
public function ship(OrderData $order): OrderData { /* ... */ }
```

→ Deep dive: [Connecting Handlers with Channels](connecting-handlers-with-channels.md)

---

## Sagas — when state must persist between events

Use when the process spans events arriving over time and you need to remember what already happened: payment received → wait → ship → notify, with hours or days between steps. A `#[Saga]` is a plain PHP class with state, an `#[Identifier]`, and event handlers. State is persisted per identifier; on the next event arrival, Ecotone rehydrates the saga from your database.

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

For full replay semantics — every state transition recorded as an event in your own database, the same durability model Temporal uses internally — use `#[EventSourcingSaga]` instead. See [Durable Execution in PHP](../../solutions/durable-execution.md#event-sourced-sagas-the-same-replay-model-in-your-own-database) for the side-by-side with Temporal.

→ Deep dive: [Sagas](sagas.md)

---

## Orchestrators — when the step list is the workflow *(Enterprise)*

Use when the workflow definition belongs in one place that a business stakeholder can read, when steps need to be reusable across multiple workflows, or when the step list is chosen dynamically from input (digital vs physical fulfillment, premium vs standard). The `#[Orchestrator]` method returns the channel list; each step is an independently testable `#[InternalHandler]`.

```php
#[Orchestrator(inputChannelName: 'order.fulfill')]
public function plan(PlaceOrder $order): array
{
    return $order->isDigital()
        ? ['order.charge', 'order.deliver_digital', 'order.notify']
        : ['order.charge', 'order.reserve_stock', 'order.ship', 'order.notify'];
}
```

→ Deep dive: [Orchestrators](orchestrators.md)

---

## How to Choose

1. **Does any state need to survive between message arrivals — does the next step depend on something that happened hours or days earlier?**
   → **Saga.** (Event-sourced if you want full replay and projections over the workflow's history.)
2. **Is the step list itself the value — written in one place a stakeholder reads, or chosen dynamically per input?**
   → **Orchestrator.**
3. **Linear pipeline where each step's output is the next step's input, and the message carries everything?**
   → **Stateless Workflow.**

The three are not exclusive — a Saga can dispatch into a Stateless Workflow, an Orchestrator step can publish events that drive a Saga. Pick the *primary* shape of the process; compose the rest around it.

## Materials

### Demo implementation

* [Synchronous Stateless Workflow](https://github.com/ecotoneframework/quickstart-examples/tree/main/Workflows/SynchronousStateless)
* [Asynchronous Stateless Workflow](https://github.com/ecotoneframework/quickstart-examples/tree/main/Workflows/AsynchronousStateless)
* [Saga (Stateful Workflow)](https://github.com/ecotoneframework/quickstart-examples/tree/main/Workflows/Saga)

### Links

* [What If 80% of Your Workflow Code Shouldn't Exist?](https://blog.ecotone.tech/three-different-ways-to-build-workflows/) \[Article]
* [Building workflows in PHP using Orchestrator](https://blog.ecotone.tech/building-workflows-in-php/) \[Article]
* [Building workflows in PHP with pipe and filter architecture](https://blog.ecotone.tech/building-workflows-in-php-with-ecotone/) \[Article]
* [Durable Execution in PHP](../../solutions/durable-execution.md) — when "durable workflows" or Temporal comes up
