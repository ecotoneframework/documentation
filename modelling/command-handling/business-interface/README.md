---
description: Business Interfaces for type-safe messaging in Ecotone PHP
---

# Business Interface

You inject `CommandBus` into a controller. Three controllers later, every call site has the magic-string `'ticket.create'` and a `new CreateTicket(...)`. The bus signature is `mixed → mixed`. There's no single place that documents what your domain can do, no IDE help, and refactoring the `CreateTicket` class is a search-and-replace across the codebase.

A **Business Interface** is your domain's typed API: `interface TicketApi { public function create(CreateTicket $cmd): TicketId; public function close(#[Identifier] string $id): void; }`. Ecotone delivers the implementation. Your controllers, console commands, and subscribers all depend on `TicketApi` — and onboarding a new developer becomes a one-file question: "what can the Ticket module do?".

This works for sending commands, executing queries, and operating on databases — keeping your application code focused on intent rather than wiring.

{% content-ref url="introduction.md" %}
[introduction.md](introduction.md)
{% endcontent-ref %}

{% content-ref url="working-with-database/" %}
[working-with-database](working-with-database/)
{% endcontent-ref %}
