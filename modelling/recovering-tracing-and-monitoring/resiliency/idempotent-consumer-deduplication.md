---
description: Idempotent consumer pattern for message deduplication in PHP
---

# Idempotency (Deduplication)

## The Problem

Stripe sends the same `payment.succeeded` webhook twice. Your handler charges the customer twice. You add a unique key on `paymentId` — except the second insert silently no-ops, so you can't tell the duplicate apart from a re-delivery. Brokers don't guarantee exactly-once delivery, so any message that has a side effect (charge, send email, increment counter) needs deduplication built in.

## How Ecotone Solves It

Ecotone stamps every Message with a unique Message Id. When Dbal Module is installed, asynchronous handlers automatically dedupe on that id — a re-delivered message is detected and skipped before your handler runs. For external messages (webhooks, third-party events), you can dedupe on a domain key — `#[Deduplicated('paymentId')]` — so your handler is invoked at most once per `paymentId` regardless of how many times the webhook fires.

## Installation

In order to use Deduplication, install [Ecotone's Dbal Module](../../../modules/dbal-support.md).

## Default Idempotent Message Consumer

In Ecotone all Messages are identifiable and contain a Message Id. The Message Id is used for deduplication by default when Dbal Module is installed. If a message was already handled, it will be skipped.&#x20;

{% hint style="success" %}
**Deduplication** is enabled by default and works whenever message is consumed in [asynchronous way](../../asynchronous-handling/).
{% endhint %}

## Custom Deduplication

You may also define given Message Handler for deduplication. This will use [Message Headers](../../../messaging/messaging-concepts/message.md) and deduplicated base on your customer header key. \
This allows in synchronous and asynchronous scenarios.&#x20;

This is especially useful when, we receive events from external services e.g. payment or notification events which contains of identifier that we may use deduplication on.\
For example Sendgrid (Email Service) sending us notifications about user interaction, as there is no guarantee that we will receive same webhook once, we may use _"eventId"_, to deduplicate in case.

```php
$this->commandBus->send($command, metadata: ["paymentId" => $paymentId]);
```

```php
final class PaymentHandler
{
    #[Deduplicated('paymentId')]
    #[CommandHandler(endpointId: "receivePaymentEndpoint")]
    public function receivePayment(ReceivePayment $command): void
    {
        // handle 
    }
}
```

`paymentId` becomes our deduplication key. Whenever we will receive now Command with same value under `paymentId` header, Ecotone will deduplicate that and skip execution of `receivePayment method`.

{% hint style="info" %}
We pass `endpointId` to the Command Handler to indicate that deduplication should happen within Command Handler with this endpoint id.\
If we would not pass that, then `endpointId` will be generated and cached automatically. This means deduplication for given Command Handler would be valid as long as we would not clear cache.
{% endhint %}

### Custom Deduplication across Handlers

Deduplication happen across given `endpointId.`This means that if we would introduce another handler with same deduplication key, it will get it's own deduplication tracking.

```php
final class PaymentHandler
{
    #[Deduplicated('paymentId')]
    #[CommandHandler(endpointId: "receivePaymentChangesEndpoint")]
    public function receivePaymentChanges(ReceivePayment $command): void
    {
        // handle 
    }
}
```

{% hint style="success" %}
As deduplication is tracked within given endpoint id, it means we can change the deduplication key safely without being in risk of receiving duplicates. If we would like to start tracking from fresh, it would be enough to change the endpointId.
{% endhint %}

## Deduplication with Expression language

We can also dynamically resolve deduplicate value, for this we can use expression language.

```php
final class PaymentHandler
{
    #[Deduplicated(expression: 'payload.paymentId')]
    #[CommandHandler(endpointId: "receivePaymentChangesEndpoint")]
    public function receivePaymentChanges(ReceivePayment $command): void
    {
        // handle 
    }
}
```

{% hint style="success" %}
**payload** variable in expression language will hold **Command/Event object. headers** variable will hold all related **Mesage Headers**.
{% endhint %}

We could also **access any object from our Dependency Container,** in order to calculate mapping:

```php
final class PaymentHandler
{
    #[Deduplicated(expression: 'reference("paymentIdMapper").map(payload.paymentId)')]
    #[CommandHandler(endpointId: "receivePaymentChangesEndpoint")]
    public function receivePaymentChanges(ReceivePayment $command): void
    {
        // handle 
    }
}
```

## Deduplication with Command Bus

Deduplicate messages at the Command Bus level to protect every handler behind that bus automatically -- without per-handler deduplication code.

**You'll know you need this when:**

* Users double-click submit buttons and create duplicate orders or payments
* Webhook providers retry delivery and your handlers process the same event twice
* Message replay during recovery causes duplicate processing
* Your handlers contain manual deduplication checks against deduplication tables

To reuse same deduplication mechanism across different Message Handlers, extend Command Bus interface with your custom one:

```php
#[Deduplicated(expression: "headers['paymentId']")]
interface PaymentCommandBus extends CommandBus
{
}
```

Then all Commands sent over this Command Bus will be deduplicated using **"paymentId"** header.

{% hint style="success" %}
This feature is available as part of **Ecotone Enterprise.**
{% endhint %}

### Command Bus name

By default using same deduplication key between Command Buses, will mean that Message will be discarded. If we want to ensure isolation that each Command Bus is tracking his deduplication separately, we can add tracking name:

```php
#[Deduplicated("paymentId", trackingName: 'payment_tracker']]
interface PaymentCommandBus extends CommandBus
{
}
```

## Deduplication clean up

To remove expired deduplication history which is kept in database table, Ecotone provides an console command:&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:deduplication:remove-expired-messages
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:deduplication:remove-expired-messages
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand("ecotone:deduplication:remove-expired-messages");
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
This command can be configured to run periodically e.g. using cron jobs.
{% endhint %}

By default Ecotone removes message id from deduplication storage **after 7 days in batches of 1000**.\
It can be customized in case of need:

```php
class DbalConfiguration
{
    #[ServiceContext]
    public function registerTransactions(): DbalConfiguration
    {
        return DbalConfiguration::createWithDefaults()
                // 100000 ms - 100 seconds
                ->withDeduplication(
                    expirationTime: 100000,
                    removalBatchSize: 1000
                );
    }
}
```

{% hint style="success" %}
It's important to keep removal batch size at small number. As deleting records may result in database index rebuild which will cause locking. Therefore small batch size will ensure our system can continue, while messages are being deleted in background.
{% endhint %}

## Disable Deduplication

As the deduplication is enabled by default, if you want to disable it then make use of **DbalConfiguration**.

```php
class DbalConfiguration
{
    #[ServiceContext]
    public function registerTransactions(): DbalConfiguration
    {
        return DbalConfiguration::createWithDefaults()
                ->withDeduplication(false);
    }

}
```
