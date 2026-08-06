---
description: Batched outbox publishing that relays thousands of database messages per second to any Message Broker
---

# High Throughput Outbox Forwarding

The [Outbox pattern](../recovering-tracing-and-monitoring/resiliency/outbox-pattern.md) commits messages together with your data changes, and a relay moves them to the Message Broker afterwards. With a standard Message Consumer that relay handles one message per poll cycle — receive, publish, acknowledge, transaction — so a busy outbox drains slowly, and the deeper the backlog grows, the slower each cycle becomes.

High Throughput Outbox Forwarding replaces that consumer with a dedicated publishing process which drains the outbox in batches straight from the database: it claims up to a configured batch size of rows in one query, groups them by their target channel, publishes whole batches at once, and removes delivered rows — all inside a single transaction. Messages relay in their stored wire format without being deserialized, so the relay stays a pure transport hop.

**You'll know you need this when:**

* Your outbox backlog grows faster than the relay drains it
* Spikes (imports, campaigns, batch jobs) push thousands of messages into the outbox at once
* You relay outboxes from multiple databases, or per tenant, and want one process to handle them

{% hint style="success" %}
High Throughput Outbox Forwarding is available as part of **Ecotone Enterprise.**
{% endhint %}

## Enabling Outbox Forwarding

Instead of a [Combined Message Channel](../recovering-tracing-and-monitoring/resiliency/outbox-pattern.md#handling-via-different-message-broker), define an `OutboxForwardingMessageChannel` — a Combined Message Channel made of exactly one Dbal backed outbox source and one target channel:

```php
#[ServiceContext]
public function orderChannels(): array
{
    return [
        OutboxForwardingMessageChannel::create(
            'orders', // reference name used on #[Asynchronous('orders')]
            DbalBackedMessageChannelBuilder::create('outbox'),
            'orderProcessing',
        ),
        AmqpBackedMessageChannelBuilder::create('orderProcessing')
            ->withHighThroughputPublishing(),
    ];
}
```

Message Handlers stay unchanged — they point at the reference name, exactly like with a Combined Message Channel:

```php
#[Asynchronous('orders')]
#[EventHandler(endpointId: 'orderWasPlaced')]
public function handle(OrderWasPlaced $event): void
{
    /** Do something */
}
```

Then run the forwarding process — by default it carries the outbox channel's name:

```bash
bin/console ecotone:run outbox
```

The source can be given as the Dbal channel builder itself (as above), which registers the outbox channel along the way — or by name, when the channel is defined elsewhere. When the target supports [Non-blocking Batched Delivery](non-blocking-batched-delivery.md), each drained batch is handed to the Broker as one batched write; otherwise messages are published one by one within the batch.

## Batch size

Each run of the forwarding process publishes one batch of up to the configured size (default 100):

```php
OutboxForwardingMessageChannel::create('orders', DbalBackedMessageChannelBuilder::create('outbox'), 'orderProcessing')
    ->withMaxForwardingBatchSize(500)
```

Larger batches raise throughput at the cost of holding the whole batch in memory within one transaction.

## Delivery guarantees and failure handling

Rows are claimed with row locks inside the forwarding transaction (`FOR UPDATE SKIP LOCKED` on PostgreSQL, claim markers on other databases), so scaled forwarding processes never publish the same message twice, and a crashed process's rows return to the outbox the moment its transaction rolls back.

What happens when delivering a message fails is decided by the failure strategy — configured on the forwarding channel, inheriting from the outbox channel when not set:

```php
OutboxForwardingMessageChannel::create('orders', DbalBackedMessageChannelBuilder::create('outbox'), 'orderProcessing')
    ->withFinalFailureStrategy(FinalFailureStrategy::STOP)
```

| Strategy | Behaviour when delivery fails |
| --- | --- |
| `RESEND` (default) / `RELEASE` | Only the failed rows return to the outbox and are retried on the next run; delivered rows of the same batch stay delivered — no duplicates |
| `IGNORE` | Failed rows are dropped from the outbox; the rest of the batch stays delivered |
| `STOP` | The exception propagates and the forwarding process stops; the batch transaction rolls back, so every row returns to the outbox |
| any + connection failure | The cycle always rolls back and claims restore instantly, so a restarted process retries the full batch cleanly |

The Error Channel is not involved — the outbox itself is the retry store, so failed messages never leave the delivery guarantee.

## One process for multiple outboxes

Forwarding channels may share a publishing process by pointing at the same endpoint id — covering outboxes living in different databases, each drained within its own transaction:

```php
#[ServiceContext]
public function relays(): array
{
    return [
        OutboxForwardingMessageChannel::create('orders', DbalBackedMessageChannelBuilder::create('ordersOutbox'), 'orderProcessing')
            ->withEndpointId('outboxPublisher'),
        OutboxForwardingMessageChannel::create('payments', DbalBackedMessageChannelBuilder::create('paymentsOutbox', 'payments_connection'), 'paymentProcessing')
            ->withEndpointId('outboxPublisher'),
    ];
}
```

```bash
bin/console ecotone:run outboxPublisher
```

## Multi-Tenancy

With [Multi-Tenancy](../../modules/dbal-support.md#multi-tenant-connections) each tenant already owns an outbox in its own database. The forwarding process picks the next tenant in round robin on every run and forwards that tenant's batch to the tenant's target — no configuration beyond the standard `MultiTenantConfiguration` is needed.

## Compile time safety

The outbox becomes purely a forwarding source, and Ecotone verifies it at configuration compile time:

* The source must be a Dbal backed Message Channel — anything else fails bootstrap
* The outbox can not be used as an execution channel or as an output channel of a Message Handler
* The outbox can not be reused inside a plain Combined Message Channel — every flow through it must be an `OutboxForwardingMessageChannel`
* Forwarding channels sharing one outbox must agree on batch size, endpoint id and failure strategy

## Throughput

Draining a 10 000 message outbox (PostgreSQL source, warmed consumer):

| Target | Total time | Speedup over message-by-message |
| --- | --- | --- |
| RabbitMQ | 0.87 s | ~88× |
| Kafka | 0.58 s | ~132× |
| Redis | 0.59 s | ~129× |
| Dbal Channel | 0.91 s | ~84× |
| Amazon SQS | 3.01 s | ~25× |

The same 10 000 messages take ~77 s with message-by-message relaying — and the per-message cost of the standard consumer grows as the backlog deepens, while batched forwarding improves with volume.
