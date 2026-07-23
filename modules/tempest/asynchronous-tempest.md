---
description: Asynchronous processing, workers, delayed messages and dead letter in Tempest
---

# Asynchronous Processing and Workers

Ecotone brings durable asynchronous processing to Tempest applications: database-backed message channels running on your existing Tempest connection, background workers driven from `./tempest`, retries with backoff, a replayable dead letter, and per-message delayed delivery.

## Durable Message Channel

Declare a channel with a `ServiceContext` — it runs on the [Tempest database connection](database-connection-dbal-module.md) your application already has:

```php
use Ecotone\Dbal\DbalBackedMessageChannelBuilder;
use Ecotone\Messaging\Attribute\ServiceContext;

final class MessagingConfiguration
{
    #[ServiceContext]
    public function notificationsChannel(): DbalBackedMessageChannelBuilder
    {
        return DbalBackedMessageChannelBuilder::create('notifications');
    }
}
```

Any handler marked `#[Asynchronous('notifications')]` now runs in the background:

```php
use Ecotone\Messaging\Attribute\Asynchronous;
use Ecotone\Modelling\Attribute\EventHandler;

final readonly class NotificationRecorder
{
    #[Asynchronous('notifications')]
    #[EventHandler(endpointId: 'notifications.order_placed')]
    public function whenOrderWasPlaced(OrderWasPlaced $event): void
    {
        // executed by the background worker
    }
}
```

{% hint style="success" %}
Because the channel is database-backed, the event message is committed in the same transaction as your Tempest model writes — the [Outbox pattern](../../modelling/recovering-tracing-and-monitoring/resiliency/outbox-pattern.md) by construction. "Order saved but event lost" cannot happen. Redelivered messages are skipped automatically through Ecotone's deduplication.
{% endhint %}

## Running the Worker

Consumers run through Tempest's own console:

```bash
./tempest ecotone:run notifications
```

Useful options for production workers:

```bash
./tempest ecotone:run notifications \
    --handledMessageLimit=100 \
    --executionTimeLimit=30000 \
    --memoryLimit=256
```

This runs a bounded cycle: at most 100 messages, at most 30 seconds, at most 256 MB — then exits. Combine it with a restart policy (supervisor, or a container `restart: unless-stopped`) and the worker recycles itself cleanly, picking up fresh code and releasing memory on every cycle. `--stopOnFailure` is available for debugging a failing consumer.

## Delayed Messages

Delayed delivery attaches a due time to one specific message — it is queued immediately and released by the channel when due, surviving worker restarts in between:

```php
use Ecotone\Messaging\Attribute\Asynchronous;
use Ecotone\Messaging\Attribute\Endpoint\Delayed;
use Ecotone\Messaging\Scheduling\TimeSpan;
use Ecotone\Modelling\Attribute\EventHandler;

#[Delayed(new TimeSpan(seconds: 30))]
#[Asynchronous('notifications')]
#[EventHandler(endpointId: 'review_request.on_shipment_dispatched')]
public function requestReview(ShipmentWasDispatched $event, Mailer $mailer): void
{
    $mailer->send(new GenericEmail(
        subject: sprintf('How was order #%s?', $event->orderId),
        to: $event->customerEmail,
        html: sprintf('<p>Hi %s, tell us how it went.</p>', $event->customerName),
    ));
}
```

Note the handler injects Tempest's own `Mailer` — Tempest services resolve directly into handler parameters, so asynchronous emails are one attribute away.

## Retries and Dead Letter

A failing handler never blocks the channel or kills the worker. Configure retries with backoff and a database-backed [Dead Letter](../../modelling/recovering-tracing-and-monitoring/resiliency/dead-letter.md) with one `ServiceContext` method:

```php
use Ecotone\Messaging\Attribute\ServiceContext;
use Ecotone\Messaging\Handler\Recoverability\ErrorHandlerConfiguration;
use Ecotone\Messaging\Handler\Recoverability\RetryTemplateBuilder;

#[ServiceContext]
public function errorHandling(): ErrorHandlerConfiguration
{
    return ErrorHandlerConfiguration::createWithDeadLetterChannel(
        'errorChannel',
        RetryTemplateBuilder::exponentialBackoff(initialDelay: 1000, multiplier: 3)
            ->maxRetryAttempts(2),
        'dbal_dead_letter',
    );
}
```

Point `defaultErrorChannel` at it in your [configuration](tempest-configuration.md):

```php
return new EcotoneConfig(
    defaultErrorChannel: 'errorChannel',
);
```

With this in place, a poison message is retried (1s, then 3s in this example) and then parked in the dead letter with its full stacktrace and payload — while other messages on the same channel keep flowing.

The dead-letter tooling is available natively in Tempest's console:

```bash
./tempest ecotone:deadletter:list          # parked messages, failure time, error
./tempest ecotone:deadletter:show <id>     # full payload and stacktrace
./tempest ecotone:deadletter:replay <id>   # push the message through again
./tempest ecotone:deadletter:replayAll
./tempest ecotone:deadletter:delete <id>
```

After deploying a fix, `replay` re-runs the parked message through its normal handler path — nothing is lost, and recovery is a console command instead of manual database surgery.

{% hint style="success" %}
All of the above runs in the [live demo application](https://github.com/ecotoneframework/tempest-ecotone-demo) — including a scripted failure drill: break a mailbox, watch the retries and the dead-letter parking, then replay the message from `./tempest`.
{% endhint %}

## Serialization of Collections

For asynchronous handling and event sourcing, messages are serialized. When using the [JMS Converter](../jms-converter.md), type collections with DTO docblocks — `@param OrderLine[] $items` on a plain readonly class — rather than array shapes (`array<array{...}>`), which the serializer does not support.
