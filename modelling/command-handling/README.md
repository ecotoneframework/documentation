---
description: PHP Message Bus, CQRS, Command Event Query Handlers
---

# Message Bus and CQRS

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

Your service classes mix reading and writing. A single change to how orders are placed breaks the order listing page. Business rules are scattered across controllers, listeners, and services — there's no clear boundary between "what changes state" and "what reads state."

## How Ecotone Solves It

Ecotone introduces **Command Handlers** for state changes, **Query Handlers** for reads, and **Event Handlers** for reactions. Each has a single responsibility, wired automatically through PHP attributes. No base classes, no framework coupling — just clear separation of concerns on top of your existing Laravel or Symfony application.

---

In this chapter we will cover process of handling and dispatching Messages with Ecotone. \
We will discuss topics like Commands, Events and Queries, Message Handlers, Message Buses, Aggregates and Sagas. \
You may be interested in theory - [DDD and CQRS](../message-driven-php-introduction.md) chapter first.

## Materials

### Demo implementation

* [Dispatching and handling Commands](https://github.com/ecotoneframework/quickstart-examples/tree/main/CQRS)
* [Dispatching and handling Events](https://github.com/ecotoneframework/quickstart-examples/tree/main/EventHandling)
* [Business Interface](https://github.com/ecotoneframework/quickstart-examples/tree/main/WorkingWithAggregateDirectly)

### Links

* [Build Symfony and Doctrine ORM Applications with ease](https://blog.ecotone.tech/build-symfony-application-with-ease-using-ecotone/) \[Article]
* [Build Laravel Application using DDD and CQRS](https://blog.ecotone.tech/build-laravel-application-using-ddd-and-cqrs/) \[Article]
* [DDD and  Message based communication with Laravel](https://blog.ecotone.tech/ddd-and-messaging-with-laravel-and-ecotone/) \[Article]
* [Going into CQRS with PHP](https://blog.ecotone.tech/cqrs-in-php/) \[Article]
* [Event Handling in PHP](https://blog.ecotone.tech/event-handling-in-php/) \[Article]
