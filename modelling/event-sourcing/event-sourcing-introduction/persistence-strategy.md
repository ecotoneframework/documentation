---
description: PHP Event Sourcing Persistence Strategy
---

# Event Stream Persistence

Once you decide to store events, you have to decide *how* — what database table holds them, how aggregate identity and version are constrained at the storage layer, what happens when you rename a class, and how PII fields are encrypted. This section covers those choices.

The pages below are mostly independent — read the one that matches the problem in front of you:

{% content-ref url="persistence-strategy/persistence-strategies.md" %}
[persistence-strategies.md](persistence-strategy/persistence-strategies.md)
{% endcontent-ref %}

How Ecotone lays out events on disk: simple append-only streams, partitioned streams (one stream per aggregate with version uniqueness), or a single global stream. Pick the one that matches your concurrency and ordering needs.

{% content-ref url="persistence-strategy/event-sourcing-repository.md" %}
[event-sourcing-repository.md](persistence-strategy/event-sourcing-repository.md)
{% endcontent-ref %}

When you already store events somewhere else (EventSauce, Patchlevel, your own table) and want Ecotone to read/write through that storage instead of its built-in event store.

{% content-ref url="persistence-strategy/making-stream-immune-to-changes.md" %}
[making-stream-immune-to-changes.md](persistence-strategy/making-stream-immune-to-changes.md)
{% endcontent-ref %}

You renamed `App\Order\Order` to `App\Sales\Order` — now production events still reference the old class. `#[NamedEvent]`, `#[Stream]`, and `#[AggregateType]` decouple stored event names from PHP class names so refactors don't break stored data.

{% content-ref url="persistence-strategy/snapshoting.md" %}
[snapshoting.md](persistence-strategy/snapshoting.md)
{% endcontent-ref %}

Your aggregate has 50,000 events and rehydration takes seconds. Snapshots store a periodic checkpoint so the aggregate loads from `snapshot + recent events`.

{% content-ref url="persistence-strategy/event-serialization-and-pii-data-gdpr.md" %}
[event-serialization-and-pii-data-gdpr.md](persistence-strategy/event-serialization-and-pii-data-gdpr.md)
{% endcontent-ref %}

Your event store contains user emails. The user requests deletion under GDPR and your store is append-only. Crypto-shredding via per-subject keys satisfies the requirement without rewriting events.
