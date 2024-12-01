# Delaying Messages

In case of Ecotone we don't delay whole Message, but specific Message Handler. \
This helps in scenarios when we have multiple Event Handler and we would like to configure the delay differently. For example may we have a case, where as a result of Order being placed, we would want to delay notification, yet to call Payment Service right away.&#x20;

{% hint style="info" %}
[Asynchronous Message Channel ](./)need to support delays, in order to make use of this feature.
{% endhint %}

## Message Handler Delay

You may delay handling given asynchronous message by adding `#[Delayed]` attribute.

```php
#[Delayed(new TimeSpan(seconds: 50))]
#[Asynchronous("notifications")]
#[EventHandler(endpointId: "welcomeEmail")]
public function sendWelcomeNotificationlWhen(UserWasRegistered $event): void
{
   // handle welcome notification
}
```

### Using Expression language

To dynamically calculate expected delay, we can use expression language.

```php
#[Delayed(expression: 'payload.dueDate']
#[Asynchronous("orders")]
#[EventHandler(endpointId: "cancelOrder")]
public function cancelOrderIfExpired(OrderWasPlaced $event): void
{
   // it will trigger at payload.dueDate, which is \DateTime object
}
```

{% hint style="success" %}
**payload** variable in expression language will hold **Command/Event object.** \
**headers** variable will hold all related **Mesage Headers**.
{% endhint %}

We could also **access any object from our Dependency Container,** in order to calculate the delay and pass there our **Command**:

```php
#[Delayed(expression: 'reference("delayService").calculate(payload)']
#[Asynchronous("orders")]
#[EventHandler(endpointId: "cancelOrder")]
public function cancelOrderIfExpired(OrderWasPlaced $event): void
{
   
}
```

## Message Delay

We may send an Message and tell Ecotone to delay it using **deliveryDelay** Message Header:

```php
$commandBus->sendWithRouting(
    "askForOrderReview", 
    "userId", 
    metadata: ["deliveryDelay" => new TimeSpan(days: 1)]
);
```

{% hint style="success" %}
If Message Delay would be send for Event. Then all subscribing Event Handlers would be delayed. For customizing it on the single Handler level, use Message Handler delay.
{% endhint %}

### Delaying using exact Date

We may also delay to given date time:

```php
$commandBus->sendWithRouting(
    "askForOrderReview", 
    "userId", 
    metadata: ["deliveryDelay" => (new \DateTimeImmutable)->modify('+1 day')]
);
```
