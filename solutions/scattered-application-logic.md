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
