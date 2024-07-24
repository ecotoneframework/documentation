---
description: Using Event Sourcing in PHP
---

# Event Sourcing Introduction

Before diving into this section be sure to understand how Aggregates works in Ecotone based on [previous sections](../../command-handling/state-stored-aggregate/).

## Difference between Aggregate Types

Ecotone provides higher level abstraction to work with Event Sourcing, which is based on Event Sourced Aggregates. Event Sourced Aggregate just like normal Aggregates protect our business rules, the difference is in how they are stored.&#x20;

### State-Stored Aggregates

Normal Aggregates are stored based on their current state:

<figure><img src="../../../.gitbook/assets/state_stored_aggregate.png" alt=""><figcaption><p>State-Stored Aggregate State</p></figcaption></figure>

Yet if we change the state,  then our previous history is lost:

<figure><img src="../../../.gitbook/assets/product_aggregate_price_change.png" alt=""><figcaption><p>Price was changed, we don't know what was the previous price anymore</p></figcaption></figure>

Having only the current state may be fine in a lot of cases and in those situation it's perfectly fine to make use of [State-Stored Aggregates](../../command-handling/state-stored-aggregate/#state-stored-aggregate). This is most easy way of dealing with changes, we change and we forget the history, as we are interested only in current state.&#x20;

{% hint style="success" %}
When we actually need to know what was the history of changes, then State-Stored Aggregates are not right path for this. If we will try to adjust them so they are aware of history we will most likely complicate our business code. This is not necessary as there is better solution - **Event Sourced Aggregates**.
{% endhint %}

### Event Sourcing Aggregate&#x20;

When we are interested in history of changes, then Event Sourced Aggregate will help us. \
Event Sourced Aggregates are stored in forms of Events. This way we preserve all the history of given Aggregate:

<figure><img src="../../../.gitbook/assets/es_product.png" alt=""><figcaption><p>Event-Sourced Aggregate</p></figcaption></figure>

When we change the state the previous Event is preserved, yet we add another one to the audit trail (Event Stream).

<figure><img src="../../../.gitbook/assets/es_price_change.png" alt=""><figcaption><p>Price was changed, yet we still have the previous Event in audit trail (Event Stream)</p></figcaption></figure>

This way all changes are preserved and we are able to know what was the historic changes of the Product.

## Event Stream

The audit trail of all the Events that happened for given Aggregate is called **Event Stream**.\
Event Stream contains of all historic Events for all instance of specific Aggregate type, for example al Events for Product Aggregate

<figure><img src="../../../.gitbook/assets/event_strea.png" alt=""><figcaption><p>Product Event Stream</p></figcaption></figure>

Let's now see how we can make use of it in the code.
