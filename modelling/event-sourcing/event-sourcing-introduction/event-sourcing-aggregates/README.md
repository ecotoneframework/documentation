---
description: "Event Sourcing Aggregates in Ecotone PHP"
---

# Event Sourcing Aggregates

## The Problem

You're a year into your Order system. Compliance asks: "show me everything that happened to order #4192, with timestamps, in order." You can't — the `orders` table only stores the current row. The audit log you bolted on is missing two columns and the listener that wrote to it crashed silently in 2023.

## How Ecotone Solves It

An **Event Sourced Aggregate** stores its history as a sequence of events (`OrderWasPlaced`, `LineItemAdded`, `PaymentReceived`, `OrderShipped`) instead of just its current state. The current state is a function of the events. The audit log isn't a separate concern — it **is** the storage. When you need a new read model (a list page, a search index, a reporting view), Projections feed off the same event stream without rerunning your handlers.

Reach for an event-sourced aggregate when the *history* matters as much as the current state — finance, healthcare, billing, regulated audit trails — or when you'll want to derive multiple read models from the same domain over time.

The pages below walk through declaring event-sourced aggregates, applying events to rebuild state, and the different ways to record events from your handlers.

{% content-ref url="working-with-aggregates.md" %}
[working-with-aggregates.md](working-with-aggregates.md)
{% endcontent-ref %}

{% content-ref url="applying-events.md" %}
[applying-events.md](applying-events.md)
{% endcontent-ref %}

{% content-ref url="different-ways-to-record-events.md" %}
[different-ways-to-record-events.md](different-ways-to-record-events.md)
{% endcontent-ref %}
