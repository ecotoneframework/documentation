---
description: Tempest CQRS DDD Event Sourcing with Ecotone
---

# Tempest

Ecotone integrates with your existing [Tempest](https://tempestphp.com) application — **Tempest active-record models** for aggregate persistence and your **Tempest database connection** reused for DBAL (outbox, event sourcing, multi-tenant). You keep Tempest's modern, attribute-first developer experience and add enterprise messaging architecture.

The same Ecotone code — aggregates, `#[CommandHandler]`, sagas, projections — runs on Tempest exactly as it does on Laravel, Symfony or Ecotone Lite.

## Installation

Follow [Installation menu](../../install-php-service-bus.md#install-for-tempest).

{% hint style="success" %}
By design, Ecotone auto-discovers Attributes in the PSR-4 roots of your Tempest application (for example `App\`) — no configuration file is required, and the `CommandBus`, `QueryBus` and `EventBus` gateways are injectable anywhere in your app.\
\
If you want to override what is scanned, provide a [namespaces](tempest-configuration.md#namespaces) configuration.
{% endhint %}

## Tempest Services in your Handlers

Handler parameters resolve from Tempest's container, including services provided through Tempest initializers — so framework services like the `Mailer` inject directly:

```php
use Tempest\Mail\GenericEmail;
use Tempest\Mail\Mailer;

#[Asynchronous('notifications')]
#[EventHandler(endpointId: 'order_confirmation.send')]
public function sendConfirmation(OrderWasPlaced $event, Mailer $mailer): void
{
    $mailer->send(new GenericEmail(
        subject: sprintf('Order #%d confirmed', $event->orderId),
        to: $event->customerEmail,
        html: '...',
    ));
}
```

Combined with a [durable channel](asynchronous-tempest.md), this is asynchronous email sending in one attribute.

## Materials

### Demo implementation

* [Tempest Projection (Database Read Model)](https://github.com/ecotoneframework/quickstart-examples/tree/main/Tempest/Projection/DatabaseReadModel)
* [Tempest Model as DDD Aggregate](https://github.com/ecotoneframework/quickstart-examples/tree/main/Tempest/Model)
