---
description: Event Sourcing PHP
---

# Event Sourcing

`Ecotone` comes with [Prooph Event Store](http://getprooph.org/) integration. This is well known and stable solution providing event storage over databases like `Postgres`, `MySQL` or `MariaDB`. \
\
`Ecotone` provides Event Sourced [Aggregates](../modelling-1.md#aggregates), which are stored as series of events. \
Thanks to that we will be able to have the whole history of what happened with specific Aggregate.\
And build up projections, that targets specific view, in order to keep our code simple, even with really complicated queries. This divide Commands from Queries (CQRS).&#x20;

## Materials

### Demo implementation

* [Implementing Event Sourcing Aggregates](https://github.com/ecotoneframework/quickstart-examples/tree/main/EventSourcing)
* [Emitting Events from Projections](https://github.com/ecotoneframework/quickstart-examples/tree/main/EmittingEventsFromProjection)
* [Working directly with Event Sourcing Aggregates](https://github.com/ecotoneframework/quickstart-examples/tree/main/WorkingWithAggregateDirectly)

### Links

* [Starting with Event Sourcing in PHP](https://blog.ecotone.tech/starting-with-event-sourcing-in-php/) \[Article]
* [Implementing Event Sourcing Application in 15 minutes](https://blog.ecotone.tech/implementing-event-sourcing-php-application-in-15-minutes/) \[Article]
