---
description: PHP Event Sourcing Projection Event Emission
---

# Emitting Events

## The Problem

You update a wallet balance projection and want to notify the user via WebSocket — but if you subscribe to the domain event directly, the user sees the old balance because the projection hasn't refreshed yet. How do you notify **after** the projection is up to date?

The challenge is timing: domain events fire before the projection processes them. If a subscriber sends a notification immediately, the user loads the page and sees stale data.

## The Solution: Emit Events from Projections

Instead of subscribing to domain events (which fire before the projection updates), subscribe to events **emitted by the projection itself** — these fire after the Read Model is up to date.

## Emit the Event

Use `EventStreamEmitter` inside your projection to emit events after updating the Read Model:

```php
#[ProjectionV2('wallet_balance')]
#[FromAggregateStream(Wallet::class)]
class WalletBalanceProjection
{
    #[EventHandler]
    public function whenMoneyWasAdded(
        MoneyWasAddedToWallet $event,
        EventStreamEmitter $eventStreamEmitter
    ): void {
        $wallet = $this->getWalletFor($event->walletId);
        $wallet = $wallet->add($event->amount);
        $this->saveWallet($wallet);

        $eventStreamEmitter->emit([
            new WalletBalanceWasChanged($event->walletId, $wallet->currentBalance)
        ]);
    }

    #[EventHandler]
    public function whenMoneyWasSubtracted(
        MoneyWasSubtractedFromWallet $event,
        EventStreamEmitter $eventStreamEmitter
    ): void {
        $wallet = $this->getWalletFor($event->walletId);
        $wallet = $wallet->subtract($event->amount);
        $this->saveWallet($wallet);

        $eventStreamEmitter->emit([
            new WalletBalanceWasChanged($event->walletId, $wallet->currentBalance)
        ]);
    }

    (...)
}
```

Emitted events are stored in the projection's own stream.

{% hint style="info" %}
Events are stored in a stream called `project_{projectionName}`. In the example above: `project_wallet_balance`.
{% endhint %}

## Subscribing to Emitted Events

After emitting, you can subscribe to these events just like any other event — in a regular event handler or even another projection:

```php
class NotificationService
{
    #[EventHandler]
    public function when(WalletBalanceWasChanged $event): void
    {
        // Send WebSocket notification — the Read Model is already up to date
    }
}
```

{% hint style="success" %}
All emitted events are stored in streams, so you can create another projection that subscribes to them — building derived views from derived views.
{% endhint %}

## Linking Events to Other Streams

In some cases you may want to emit an event to an existing stream (for example, to provide a summary event) or to a custom stream:

```php
$eventStreamEmitter->linkTo('wallet', [
    new WalletBalanceWasChanged($event->walletId, $wallet->currentBalance)
]);
```

{% hint style="success" %}
`linkTo` works from any place in the code. `emit` stores events in the projection's own stream and only works inside a projection.
{% endhint %}

## Controlling Event Emission

### During Rebuild

When a projection is rebuilt (reset and replayed from the beginning), emitted events could be republished — causing duplicate notifications and duplicate linked events.

Ecotone handles this automatically: **events emitted during a reset/rebuild phase are not republished or stored**. This is safe by default.

{% hint style="warning" %}
This is the key difference between using `EventStreamEmitter` versus `EventBus`. The `EventBus` would simply republish events during a rebuild, causing duplicates. `EventStreamEmitter` suppresses them.
{% endhint %}

### With ProjectionDeployment (Enterprise)

You can also explicitly suppress event emission by setting `live: false` on `#[ProjectionDeployment]`:

```php
#[ProjectionV2('wallet_balance')]
#[FromAggregateStream(Wallet::class)]
#[ProjectionDeployment(live: false)]
class WalletBalanceProjection
{
    // EventStreamEmitter calls are silently skipped — no events are stored or published
}
```

This is important because **backfill will emit events** — it replays historical events through your handlers, and if those handlers call `EventStreamEmitter`, all those events will be published to downstream consumers. If you're backfilling a projection with 2 years of history, that means thousands of duplicate notifications.

Use `live: false` during backfill to prevent this, then switch to `live: true` once the projection is caught up. This is the pattern used in [blue-green deployments](blue-green-deployments.md).

{% hint style="info" %}
`#[ProjectionDeployment]` is available as part of Ecotone Enterprise.
{% endhint %}

## Deleting the Projection

When a projection is deleted, Ecotone automatically deletes the projection's event stream (`project_{name}`).

{% hint style="info" %}
Custom streams created via `linkTo` are not automatically deleted — they may be shared with other consumers.
{% endhint %}

## Demo

[Example implementation using Ecotone Lite.](https://github.com/ecotoneframework/quickstart-examples/tree/master/EmittingEventsFromProjection)
