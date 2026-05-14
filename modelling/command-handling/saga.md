---
description: Process Manager Saga PHP
---

# Saga Introduction

You're handling order processing: charge the card, reserve stock, schedule shipping, send a confirmation. The steps span hours or days, run across services, and any one of them can fail. State columns like `is_payment_processed`, `retry_count`, `step_completed_at` start appearing on the Order table — and the workflow ends up tangled across listeners, jobs, and cron sweepers.

A **Saga** is a class that owns a long-running business process. It listens for events, decides what to do next, dispatches commands, and persists its own state between steps. Where an Aggregate protects the invariants of a single thing, a Saga coordinates the steps that move many things forward.

```php
#[Saga]
class OrderFulfillment
{
    #[Identifier]
    private string $orderId;
    private OrderState $state;

    #[EventHandler]
    public static function start(OrderWasPlaced $event): self { /* ... */ }

    #[EventHandler]
    public function onPaymentReceived(PaymentReceived $event, CommandBus $bus): void
    {
        $bus->send(new ReserveStock($this->orderId));
    }
}
```

## Saga vs Aggregate

Both are message-driven persisted classes. The distinction is what they own:

- An **Aggregate** owns business **state and invariants** for one entity (an Order, an Account, a Subscription) — its handlers protect rules about that entity.
- A **Saga** owns a business **process** — its handlers track progress across many entities or services, often over time, and react to events as they arrive.

If you need to wait for something, branch on the result, time out after 24 hours, or coordinate across multiple aggregates, that's a Saga.

## Where to go next

Sagas are part of Ecotone's broader Business Workflow support, alongside output-channel chaining and Orchestrators. The full implementation guide — including delayed timeouts, branching, failure handling, and a worked Order/Payment/Shipping example — lives in:

{% content-ref url="../business-workflows/sagas.md" %}
[sagas.md](../business-workflows/sagas.md)
{% endcontent-ref %}

For deciding between a Saga and the simpler workflow primitives:

{% content-ref url="../business-workflows/" %}
[business-workflows](../business-workflows/)
{% endcontent-ref %}
