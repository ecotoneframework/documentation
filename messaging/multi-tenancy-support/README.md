---
description: Multi-Tenancy Ecotone, Symfony, Laravel, DDD, CQRS, Event Sourcing
---

# Multi-Tenancy Support

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

Adding a new tenant means configuring new queues, new database connections, and custom routing. A slow tenant's queue backlog delays everyone else's messages. Your multi-tenancy logic is tangled into business code instead of being handled at the infrastructure level.

## How Ecotone Solves It

Ecotone provides **built-in multi-tenancy** where the code you write is the same as in non-multi-tenant systems. Tenant context is propagated automatically through messages. Your business code doesn't need to know about tenants — Ecotone handles routing, isolation, and tenant switching at the messaging layer.

---

Ecotone provides support for building Multi-Tenant based Systems. As Ecotone aims to focus on the business part of the system, not technical parts, the code we will write will be the same as in non Multi-Tenant systems.\
This way even if our System does not need Multi-Tenancy now, if we will consider it in future, it will work out of the box. And we can safely focus on the business part of the system.

## Materials

## Demo

* [Example implementation of different scenario for Multi-Tenancy](https://github.com/ecotoneframework/quickstart-examples/tree/main/MultiTenant)

## Links

* [Symfony Multi-Tenant Applications](https://blog.ecotone.tech/symfony-multi-tenant-applications-with-ecotone/)
* [Laravel Multi-Tenant Applications](https://blog.ecotone.tech/laravel-multi-tenant-systems-with-ecotone/)

