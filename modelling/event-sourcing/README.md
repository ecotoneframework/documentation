---
description: Event Sourcing PHP
---

# Event Sourcing

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

You store the current state of your entities, but not how they got there. When a customer disputes a charge, you can't answer "what exactly happened?" Rebuilding read models after a schema change means writing migration scripts by hand. Auditors ask for a complete trail of changes and you piece it together from application logs.

## How Ecotone Solves It

Ecotone provides **Event Sourcing** as a first-class feature. Instead of storing current state, you store the sequence of events that led to it. Rebuild any view of the data by replaying events. Get a complete, immutable audit trail automatically. Works with **Postgres**, **MySQL**, and **MariaDB** for event storage, with projections that can write to any storage you choose.

---

Read more in the following chapters.

## Materials

### Demo implementation

* [Implementing Event Sourcing Aggregates](https://github.com/ecotoneframework/quickstart-examples/tree/main/EventSourcing)
* [Emitting Events from Projections](https://github.com/ecotoneframework/quickstart-examples/tree/main/EmittingEventsFromProjection)
* [Working directly with Event Sourcing Aggregates](https://github.com/ecotoneframework/quickstart-examples/tree/main/WorkingWithAggregateDirectly)

### Links

* [Starting with Event Sourcing in PHP](https://blog.ecotone.tech/starting-with-event-sourcing-in-php/) \[Article]
* [Implementing Event Sourcing Application in 15 minutes](https://blog.ecotone.tech/implementing-event-sourcing-php-application-in-15-minutes/) \[Article]
