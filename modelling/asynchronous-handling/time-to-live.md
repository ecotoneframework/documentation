# Time to Live

We may define Time to Live for Messages. This way if Message will not be handled within specific amount of time, it will be automatically discarded. \
This is useful in scenarios like sending notifications, where given notification like One Time Password may actually have meaning only for 5 minutes.&#x20;

## Message Handler Time to Live

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

### Using Expression language

To dynamically calculate expected TTL, we can use expression language.

```php
#[TimeToLive(expression: 'headers["expirationTime"]']
#[Asynchronous("notifications")]
#[EventHandler(endpointId: "welcomeEmail")]
public function sendWelcomeNotificationWhen(UserWasRegistered $event): void
{
   // handle welcome notification
}
```

{% hint style="success" %}
**payload** variable in expression language will hold **Command/Event object.** \
**headers** variable will hold all related **Mesage Headers**.
{% endhint %}

We could also **access any object from our Dependency Container,** in order to calculate the delay and pass there our **Command**:

```php
#[Delayed(expression: 'reference("delayService").calculate(payload)']
#[Asynchronous("notifications")]
#[EventHandler(endpointId: "welcomeEmail")]
public function sendWelcomeNotificationWhen(UserWasRegistered $event): void
{
   
}
```

## Message Time to Live

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
