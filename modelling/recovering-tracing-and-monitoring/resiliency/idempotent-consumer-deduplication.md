# Idempotent Consumer (Deduplication)

## Installation

In order to use Deduplication, install [Ecotone's Dbal Module](../../../modules/dbal-support.md).

## Deduplication (Idempotent Consumer)

The role of deduplication is to safely receive same message multiple times, as there is no guarantee from Message Brokers that we will receive the same Message once.\
In Ecotone all Messages are identifiable and contains of Message Id. Message Id is used for deduplication.\
If message was already handled, it will be skipped.&#x20;

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

## Custom remove time

By default Ecotone removes message id from deduplication storage after 7 days. \
It can be customized in case of need:

```php
class DbalConfiguration
{
    #[ServiceContext]
    public function registerTransactions(): DbalConfiguration
    {
        return DbalConfiguration::createWithDefaults()
                // 10000 ms - 10 seconds
                ->withDeduplication(true, 10000);
    }

}
```

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
