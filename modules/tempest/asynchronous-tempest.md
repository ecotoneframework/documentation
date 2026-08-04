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
#[EventHandler(endpointId: 'review_request.on_shipment_dispatched', outputChannelName: 'notification.enrich')]
public function requestReview(ShipmentWasDispatched $event): EmailNotification
{
    return new EmailNotification(
        orderId: $event->orderId,
        subject: sprintf('How was order #%s?', $event->orderId),
        html: '<p>Your package is on its way. Tell us how it went.</p>',
    );
}
```

The handler composes only the content — enrichment and sending come from the pipeline it targets with `outputChannelName` (see below).

## Enriching Messages on the Way

Pipeline steps can enrich the message HEADERS while the payload passes through untouched — with `changingHeaders: true`, the returned array is merged into the headers. This keeps event handlers content-only: the recipient's account details are added where they are known, and the send step is one prepared building block any notification can flow through:

```php
final readonly class AccountDetailsEnricher
{
    #[InternalHandler(
        inputChannelName: 'notification.enrich',
        outputChannelName: 'email.send',
        changingHeaders: true,
    )]
    public function enrich(EmailNotification $notification): array
    {
        $order = Order::findById((int) $notification->orderId);

        return [
            'customerEmail' => $order->customer_email ?? '',
            'customerName' => $order->customer_name ?? '',
        ];
    }
}
```

The send step reads the payload plus the enriched headers with `#[Header]` parameters — and injects Tempest's own `Mailer`, since Tempest services resolve directly into handler parameters:

```php
use Ecotone\Messaging\Attribute\Parameter\Header;

#[InternalHandler(inputChannelName: 'email.send')]
public function send(
    EmailNotification $notification,
    #[Header('customerEmail')] ?string $customerEmail,
    #[Header('customerName')] ?string $customerName,
    Mailer $mailer,
): void {
    $mailer->send(new GenericEmail(
        subject: $notification->subject,
        to: $customerEmail,
        html: sprintf('<p>Hi %s!</p>', $customerName) . $notification->html,
    ));
}
```

Headers do not only come from enrichers. Metadata passed to the bus travels with the message, and Ecotone propagates it to the events a handler records and onward to the (asynchronous) handlers those events reach:

```php
$this->commandBus->send(
    new PlaceOrder(/* ... */),
    ['simulateEmailFailure' => true],
);
```

A step far downstream can then read it as a typed `#[Header]` parameter — `#[Header('simulateEmailFailure')] ?bool $simulateEmailFailure` — while none of the steps in between mention it. This is how request-scoped context (a correlation id, a tenant, a feature flag) reaches a background worker without being threaded through every payload on the way.

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

The same operations are available in your own application code — `DeadLetterGateway` can be injected into any Tempest controller or service, so a "parked messages" page with replay and delete buttons is a few lines on top of the interface the console commands use:

```php
use Ecotone\Dbal\Recoverability\DeadLetterGateway;

final readonly class AlertsController
{
    public function __construct(private DeadLetterGateway $deadLetter) {}

    #[Get('/alerts')]
    public function index(): View
    {
        return view('./alerts.view.php', errors: $this->deadLetter->list(limit: 50, offset: 0));
    }

    #[Post('/alerts/{messageId}/replay')]
    public function replay(string $messageId): Redirect
    {
        $this->deadLetter->reply($messageId);

        return new Redirect('/alerts');
    }
}
```

Each entry is an `ErrorContext` carrying the message id, failure timestamp, exception class and message, file, line and stacktrace — enough for an operations screen without touching the database. `count()`, `show()`, `replyAll()`, `delete()` and `deleteAll()` complete the interface.

A replayed message is not indistinguishable from a fresh one: Ecotone marks it with the `ecotone.dlq.message_replied` header, which a handler can read like any other header. That is useful when recovery should behave differently from the first attempt — skipping a step that already succeeded, relaxing a guard, or tagging the outcome as a recovery:

```php
use Ecotone\Messaging\Handler\Recoverability\ErrorContext;

#[InternalHandler(inputChannelName: 'email.send')]
public function send(
    EmailNotification $notification,
    #[Header(ErrorContext::DLQ_MESSAGE_REPLIED)] ?string $replayed,
    Mailer $mailer,
): void {
    // $replayed is '1' when this message came back from the dead letter
}
```

{% hint style="success" %}
All of the above runs in the [live demo application](https://github.com/ecotoneframework/tempest-ecotone-demo) — including a scripted failure drill: break a mailbox, watch the retries and the dead-letter parking, then replay the message from `./tempest`.
{% endhint %}

## Serialization of Collections

For asynchronous handling and event sourcing, messages are serialized. When using the [JMS Converter](../jms-converter.md), type collections with DTO docblocks — `@param OrderLine[] $items` on a plain readonly class — rather than array shapes (`array<array{...}>`), which the serializer does not support.
