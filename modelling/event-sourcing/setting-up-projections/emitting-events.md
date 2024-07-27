# Emitting events

One of the drawback of Event Sourcing is eventual consistency. \
Whenever event happens we want to do two things, update our projection and inform the end user about change. However user need to be informed about the change after projection is refreshed, as otherwise he will get stale view. \
\
In order to solve this drawback Ecotone brings possibility for emitting events directly from projection. So instead of subscribing to Domain Events (Aggregate State Changed), end user may subscribe to change in the projection.

## Emit the event

```php
#[Projection( "wallet_balance", Wallet::class)]
final class WalletBalanceProjection
{
    #[EventHandler]
    public function whenMoneyWasAdded(MoneyWasAddedToWallet $event, EventStreamEmitter $eventStreamEmitter): void
    {
        $wallet =  $this->getWalletFor($event->walletId);
        $wallet = $wallet->add($event->amount);
        $this->saveWallet($wallet);

        $eventStreamEmitter->emit([new WalletBalanceWasChanged($event->walletId, $wallet->currentBalance)]);
    }

    #[EventHandler]
    public function whenMoneyWasSubtract(MoneyWasSubtractedFromWallet $event, EventStreamEmitter $eventStreamEmitter): void
    {
        $wallet =  $this->getWalletFor($event->walletId);
        $wallet = $wallet->subtract($event->amount);
        $this->saveWallet($wallet);

        $eventStreamEmitter->emit([new WalletBalanceWasChanged($event->walletId, $wallet->currentBalance)]);
    }

    (...)
}
```

In order to emit the events, we are using `EventStreamEmitter`.\
Whenever we `emit` given events, they are stored in Projection's stream.

{% hint style="info" %}
Events are stored in stream called project_{projectionName}._ In above case it will be _project\_wallet\_balance._
{% endhint %}

After that you may subscribe to given events, just like to any other events.

```php
final class NotificationService
{
    #[EventHandler]
    public function when(WalletBalanceWasChanged $event): void
    {
        // sending websocket event
    }
}
```

{% hint style="success" %}
All the events are stored in the streams, this means that in case of need we may create another projection that will subscribe to those events.
{% endhint %}

## Linking Events

In some cases we may want to emit event to existing stream (for example to provide summary event), or to fresh new stream.\
In order to do that we may use `linkTo` method on `EventStreamEmitter`.

```php
$eventStreamEmitter->linkTo("wallet", [new WalletBalanceWasChanged($event->walletId, $wallet->currentBalance)]);
```

{% hint style="success" %}
`LinkTo` works from any place in the code, however `emit` as it stores in projection's stream works only inside projection.
{% endhint %}

## Rebuilding the projection

When we rebuild the projection events could be republished and that would affect our end users, plus would link duplicated events to our stream. \
Luckily Ecotone will handle this scenario and will not republish or store any events that are emitted during reset phase.&#x20;

{% hint style="warning" %}
This is the biggest difference between using `EventBus` versus `EventStreamEmitter`.\
As EventBus would simple republish the events during rebuild phase.
{% endhint %}

## Deleting the projection

When projection is deleted, Ecotone will automatically delete projection stream.

{% hint style="info" %}
In case when custom stream is provided by [linking events](emitting-events.md#linking-events) they will not be automatically deleted.
{% endhint %}

## Demo

[Example implementation using Ecotone Lite.](https://github.com/ecotoneframework/quickstart-examples/tree/master/EmittingEventsFromProjection)\
