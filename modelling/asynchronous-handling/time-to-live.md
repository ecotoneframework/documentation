# Time to Live

We may define Time to Live for Messages. This way if Message will not be handled within specific amount of time, it will be automatically discarded. \
This is useful in scenarios like sending notifications, where given notification like One Time Password may actually have meaning only for 5 minutes.&#x20;

## Static Time to Live

You may delay handling given asynchronous message by adding `#[Delayed]` attribute.

```php
#[TimeToLive(new TimeSpan(seconds: 50))]
#[Asynchronous("notifications")]
#[EventHandler(endpointId: "welcomeEmail")]
public function sendWelcomeNotificationWhen(UserWasRegistered $event): void
{
   // handle welcome notification
}
```

The delay is defined in milliseconds.

## Dynamic Time to Live

We may send an Message and tell Ecotone to delay it using **deliveryDelay** Message Header:

```php
$commandBus->sendWithRouting(
    "sendOneTimePassword", 
    "userId", 
    metadata: ["timeToLive" => new TimeSpan(minutes: 5)]
);
```

{% hint style="info" %}
[Asynchronous Message Channel ](./)need to support this option to be used
{% endhint %}
