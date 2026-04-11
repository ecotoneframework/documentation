# About

<figure><img src=".gitbook/assets/ecotone_logo_no_background (1).png" alt="" width="563"><figcaption></figcaption></figure>

## CQRS, Event Sourcing, Workflows, and Production Resilience for Laravel and Symfony

Add production-grade architecture to your existing framework with a single Composer package. No framework change. No base classes. Just PHP attributes.

---

## What problem does Ecotone solve?

### Your app grew, your architecture didn't

Controllers doing too much. Business logic scattered across listeners, services, and middleware. Every change risks breaking something unrelated. Testing requires bootstrapping the entire framework.

**Ecotone introduces clear separation** — Command Handlers for writes, Query Handlers for reads, Event Handlers for reactions. Each has a single responsibility, wired automatically through PHP attributes.

### Async processing is fragile

Failed jobs disappear silently. There's no retry strategy beyond "try again 3 times." You can't replay failed messages or see what's stuck in the queue. Going async required touching every handler.

**Ecotone provides production resilience** — automatic retries, error channels, dead letter queues, outbox pattern, and idempotency. All declarative via attributes, with a single attribute to make any handler async.

### Enterprise patterns feel out of reach in PHP

You've seen CQRS, Event Sourcing, and Sagas in Java and .NET ecosystems. Every PHP implementation requires rewriting your application or adopting an opinionated framework.

**Ecotone brings these patterns to your existing stack** — works with Laravel (Eloquent, Queues, Octane) and Symfony (Doctrine, Messenger Transport) without requiring a framework change. No base classes to extend, no interfaces to implement.

---

## Choose your path

**Laravel developers** — Keep Eloquent, add enterprise messaging.\
Start with [Laravel Quick Start](quick-start-php-ddd-cqrs-event-sourcing/laravel-ddd-cqrs-demo-application.md) or [Laravel Module docs](modules/laravel/).

**Symfony developers** — Go beyond Symfony Messenger.\
Start with [Symfony Quick Start](quick-start-php-ddd-cqrs-event-sourcing/symfony-ddd-cqrs-demo-application/) or [Symfony Module docs](modules/symfony/).

**Other PHP frameworks** — Use Ecotone with any PHP application.\
Start with [Ecotone Lite docs](modules/ecotone-lite/).

**Architects** — PHP can do what Spring and Axon do.\
Read [Why Ecotone?](why-ecotone.md) to understand the positioning.

---

## How it works

1. **Install via Composer** — `composer require ecotone/laravel` or `ecotone/symfony-bundle`
2. **Add attributes to your code** — Mark methods as Command Handlers, Event Handlers, or Queries using PHP attributes
3. **Ecotone wires the messaging** — Message buses, async channels, retries, and event sourcing are handled automatically

---

## Get started in minutes

* [Install](install-php-service-bus.md) for Symfony, Laravel, or any PHP framework
* [Learn by example](quick-start-php-ddd-cqrs-event-sourcing/) — Send your first command in 5 minutes
* [Go through tutorial](tutorial-php-ddd-cqrs-event-sourcing/) — Build a complete messaging flow step by step
* [Workshops, Support, Consultancy](other/contact-workshops-and-support.md) — Hands-on training for your team

{% hint style="info" %}
Built on [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/). Used in production by teams running multi-tenant, event-sourced systems at scale.
{% endhint %}

{% hint style="success" %}
Join [Ecotone's Community Channel](https://discord.gg/GwM2BSuXeg), and ask questions there.
{% endhint %}
