# Message Priority

In case of Ecotone we don't prioritize whole Message, but specific Message Handler. \
This helps in scenarios when we have multiple Event Handlers, and we would like to configure each of them differently. For example may we have a case, where we want to prioritzed one notification over another.

{% hint style="info" %}
In some Message Brokers, it may required to enable priority support for given Message Channel.
{% endhint %}

## Static Priority

You may prioritize handling given asynchronous message by adding `#[Priority]` attribute.

```php
#[Priority(1)]
#[Asynchronous("notifications")]
#[EventHandler(endpointId: "welcomeEmail")]
public function sendWelcomeNotificationlWhen(UserWasRegistered $event): void
{
   // handle welcome notification
}

#[Priority(5)]
#[Asynchronous("notifications")]
#[EventHandler(endpointId: "welcomeEmail")]
public function sendWelcomeNotificationlWhen(UserWasRegistered $event): void
{
   // handle welcome notification
}
```

## Dynamic Priority

We may send an Message and tell Ecotone to prioritize it using **priority** Message Header:

```php
$commandBus->sendWithRouting(
    "askForOrderReview", 
    "userId", 
    metadata: ["priority" => 100]
);
```

{% hint style="info" %}
[Asynchronous Message Channel ](./)need to support this option to be used
{% endhint %}
