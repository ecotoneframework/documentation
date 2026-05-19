---
description: Outbox pattern for atomic message publishing with database transactions
---

# Outbox Pattern

## The Problem

You commit an order to the database, then publish `OrderWasPlaced` to RabbitMQ. The DB commit succeeds; the broker publish fails — and downstream services never learn the order exists. Or vice versa: the publish succeeds but the DB transaction rolls back, and you've published an event for an order that doesn't exist. Either way, your data and your messages are out of sync, and the bug is invisible until a customer calls support.

## How Ecotone Solves It

The **Outbox pattern** writes the message to the *same* database transaction as the data change. If the transaction commits, the message is durable — Ecotone publishes it asynchronously after commit. If it rolls back, the message vanishes with the rest of the change. There is no two-storage atomicity problem because there is only one storage.

{% hint style="success" %}
For critical parts of the system — payments, payouts, audit trails — commit messages to the same database as the data changes using Outbox. Lower-stakes notifications can publish directly to the broker.
{% endhint %}

{% hint style="success" %}
**Bigger picture**: The Outbox pattern is one of Ecotone's [Durable Execution](../../../solutions/durable-execution.md) primitives — together with sagas, retries, and channel redelivery, it gives crash-resistant processes on the database and broker you already run, with no separate workflow service or Temporal cluster.
{% endhint %}

## Installation

In order to use Outbox pattern we need to set up [Dbal Module](../../../modules/dbal-support.md#using-existing-connection).

### **Dbal Message Channel**

By sending asynchronous messages via database, we are storing them together with data changes. This thanks to default transactions for Command Handlers, commits them together.

```php
#[ServiceContext]
public function databaseChannel()
{
    return DbalBackedMessageChannelBuilder::create("async");
}
```

### **Asynchronous Event Handler**

```php
#[Asynchronous("async")]
#[EventHandler(endpointId:"notifyAboutNeworder")]
public function notifyAboutNewOrder(OrderWasPlaced $event) : void
{
    // notify about new order
}
```

\
After this all your messages will be go through your database as a message channel.&#x20;

## Setup Outbox where it's needed

With Ecotone's Outbox pattern we set up given Channel to run via Database. This means that we can target specific channels, that are crucial to run under outbox pattern. \
In other cases where data consistency is not so important to us, we may actually use Message Broker Channels directly and skip the Outbox. \
\
As an example, registering payments and payouts may an crucial action in our system, so we use it with Outbox pattern. However sending an "Welcome" notification may be just fine to run directly with Message Broker.

## Scaling the solution

One of the challenges of implementing Outbox pattern is way to scale it. When we start consume a lot of messages, we may need to run more consumers in order to handle the load.

### Publishing deduplication

In case of Ecotone, you may safely scale your [Messages Consumers](../../microservices-php/message-consumer.md) that are consuming from your `Dbal Message Channel`. Each message will be reserved for the time of being published, thanks to that no duplicates will be sent when we scale.

### Handling via different Message Broker

However we may actually want to avoid scaling our Dbal based Message Consumers to avoid increasing the load on the database.\
For this situation `Ecotone` allows to make use so called `Combined Message Channels`.\
In that case we would run `Database Channel` only for the `outbox` and for actual `Message Handler` execution a different one.\
This is powerful concept, as we may safely produce messages with outbox and yet be able to handle and scale via `RabbitMQ` `SQS` `Redis` etc.&#x20;

```php
#[Asynchronous(["database_channel", "rabbit_channel"])]
#[EventHandler(endpointId: 'orderWasPlaced')]
public function handle(OrderWasPlaced $event): void
{
    /** Do something */
}
```

* `database_channel` is Dbal Message Channel
* `rabbit_channel` is our RabbitMQ Message Channel

Then we run one or few Message Consumers for `outbox` and we scale Message Consumers for `rabbit`.

### Combined Message Channels with reference

If we want more convient way as we would like to apply `combined message channels` on multiple Message Handlers, we may create an `reference`.

```php
#[ServiceContext]
public function combinedMessageChannel(): CombinedMessageChannel
{
    return CombinedMessageChannel::create(
        'outbox_sqs', //Reference name
        ['database_channel', 'amazon_sqs_channel'], // list of combined message channels
    );
}
```

And then we use `reference` for our `Message Handlers`.

```php
#[Asynchronous(["outbox_sqs"])]
#[EventHandler(endpointId: 'orderWasPlaced')]
public function handle(OrderWasPlaced $event): void
{
    /** Do something */
}
```
