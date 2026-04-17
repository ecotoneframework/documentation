---
description: Common challenges Ecotone solves for Laravel and Symfony developers
---

# Solutions

Every feature in Ecotone exists to solve a real problem that PHP developers face as their applications grow. Find the situation that matches yours:

| If you recognize this... | See |
|---|---|
| Business logic is scattered across controllers, services, and listeners — nobody can explain end-to-end what happens when an order is placed | [Scattered Application Logic](scattered-application-logic.md) |
| Queue jobs fail silently, a retry re-fires handlers that already succeeded, or a duplicate webhook double-charges the customer | [Unreliable Async Processing](unreliable-async-processing.md) |
| A multi-step process lives across event listeners, cron jobs, and `is_processed` columns — adding or reordering a step means editing many files | [Complex Business Processes](complex-business-processes.md) |
| Support asks "what exactly happened to this order?" and the trail is in logs, timestamps, and hope | [Audit Trail & State Rebuild](audit-trail-and-state-rebuild.md) |
| Services talk over HTTP with custom retry logic per pair, and one service going down cascades into the others | [Microservice Communication](microservice-communication.md) |
| You're evaluating whether PHP can carry enterprise architecture, with the alternative being a rewrite in Java or .NET | [PHP for Enterprise Architecture](php-for-enterprise-architecture.md) |

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}
