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

#[Asynchronous('async')]
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
