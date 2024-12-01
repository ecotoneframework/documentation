# Message Priority

In case of Ecotone we don't prioritize whole Message, but specific Message Handler. \
This helps in scenarios when we have multiple Event Handlers, and we would like to configure each of them differently. For example may we have a case, where we want to prioritized one notification over another.

In Ecotone the higher the priority, the quicker given Event Handler will be called.&#x20;

## Synchronous Prioritization

In case we publish an Event, we may want have multiple subscribing Event Handlers. In some situations we may want given action to happen before the other. \
This may happen for example for example, when one Event Handler updates an data, which the other is using. Therefore we may need to ensure that the Handler modifying data will be called before the one that make use of it.&#x20;

```php
final class UserRegisteredSubscriber
{
    #[Priority(5)]
    #[EventHandler]
    public function updateSomeData(UserWasRegistered $event): void
    {
        // updating data will be called first
    }

    #[Priority(1)]
    #[EventHandler]
    public function sendNotification(UserWasRegistered $event): void
    {
        // sending notification will be called second
    }
}
```

## Default Synchronous Prioritization

There may be cases when we will use Synchronous Projections and then we would like to use this data in Event Handler. In that case, if standard Event Handler would be called first, we would lack the data. \
Therefore by default when Standard Event Handler has same priority as Projection Event Handler, Projection will be called first.&#x20;

```php
#[Projection]
final class UserProjection
{
    #[EventHandler]
    public function handle(UserWasRegistered $event): void
    {
        // Projection will be called first
    }
}

final class UserRegisteredSubscriber
{
    #[EventHandler]
    public function sendNotification(UserWasRegistered $event): void
    {
        // sending notification will be called second
    }
}
```

This also works for Aggregates to be prioritized before Standard Event Handlers. Therefore the ordering when same priority level is used, is as follows:

1. Projection Event Handlers
2. Aggregate/Sagas Event Handlers
3. Standard Event Handlers

## Asynchronous Prioritization

Ecotone does allow to prioritize an handling of given Message before another one. \
This way we can handle quicker Messages that have been published to Asynchronous Message Channel later. \
For example we may send a lot of different notifications, however when Customer asks for One-Time Password we want to deliver it immediately. Therefore for this scenario we would setup higher priority for Authentication Token notification handling, than for other Notifications.

{% hint style="info" %}
[Asynchronous Message Channel ](./)need to support this option to be used
{% endhint %}

## Message Handler Priority

You may prioritize handling given asynchronous message by adding `#[Priority]` attribute.

```php
#[Priority(1)]
#[Asynchronous("notifications")]
#[EventHandler(endpointId: "welcomeEmail")]
public function sendWelcomeNotificationlWhen(UserWasRegistered $event): void
{
   // handle welcome notification with lower priority
}

#[Priority(5)]
#[Asynchronous("notifications")]
#[CommandHandler(endpointId: "welcomeEmail")]
public function sendAuthenticationToken(RequestToken $command): void
{
   // send authentication token quicker
}
```

## Message Priority

We may send an Message and tell Ecotone to prioritize it using **priority** Message Header:

```php
$commandBus->sendWithRouting(
    "askForOrderReview", 
    "userId", 
    metadata: ["priority" => 100]
);
```
