# Delaying Messages

In case of Ecotone we don't delay whole Message, but specific Message Handler. \
This helps in scenarios when we have multiple Event Handler and we would like to configure the delay differently. For example may we have a case, where as a result of Order being placed, we would want to delay notification, yet to call Payment Service right away.&#x20;

## Static Delay

You may delay handling given asynchronous message by adding `#[Delayed]` attribute.

```php
#[Delayed(50000)]
#[Asynchronous("notifications")]
#[EventHandler(endpointId: "welcomeEmail")]
public function sendWelcomeNotificationlWhen(UserWasRegistered $event): void
{
   // handle welcome notification
}
```

## Dynamic Delay

We may send an Message and tell Ecotone to delay it using **deliveryDelay** Message Header:

```php
$commandBus->sendWithRouting(
    "askForOrderReview", 
    "userId", 
    metadata: ["deliveryDelay" => 3 * 24 * 60 * 60 * 1000]
);
```
