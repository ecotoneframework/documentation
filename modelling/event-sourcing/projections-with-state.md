# Projections with State

In order to optimize projections or to avoid using external storage, we may use of `Projection's State`. \
State is data that is kept between executions and can be passed to Projection's Event Handler.

{% hint style="info" %}
This is also really helpful in case we want to build throw away projection, which is suppose to calculate the result, [emit the event](emitting-events.md#emit-the-event) and then to be [deleted](executing-and-managing/projection-cli-actions.md#deleting-the-projection).
{% endhint %}

## Passing state inside Projection

In order to pass the state to Projection's Event Handlers we need to mark method parameter with `#[ProjectionState]`.&#x20;

```php
#[Projection(self::NAME, Ticket::class)]
final class TicketCounterProjection
{
    const NAME = "ticket_counter";

    #[EventHandler]
    public function when(TicketWasRegistered $event, #[ProjectionState] TicketCounterState $state, EventStreamEmitter $eventStreamEmitter): TicketCounterState
    {
        $state = $state->increase();

        $eventStreamEmitter->emit([new TicketCounterWasChanged($state->count)]);

        return $state;
    }
}
```

Ecotone will resolve this parameter and pass the state. \
The returned state from the Event Handler will becomes new state for next execution.\
We may pass the state between all Event Handlers in given Projection.

{% hint style="success" %}
The state can be simple array or a class. Whatever you pick, Ecotone will automatically serialize and deserialize it for you.
{% endhint %}

## Fetching the state from outside

You may want to fetch the state from outside to return it to the end user. \
In that case Ecotone brings `ProjectionStateGateway`.

```php
interface TicketCounterGateway
{
    #[ProjectionStateGateway(TicketCounterProjection::NAME)]
    public function getCounter(): TicketCounterState;
}
```

The first parameter of the attribute is the projection name, so Ecotone can know, which state it should look for.\
This [Gateway](../../messaging/messaging-concepts/messaging-gateway.md) will automatically convert the state to your defined return type.&#x20;

{% hint style="success" %}
Gateways are automatically registered in your Dependency Container, so you can fetch it like any other service.
{% endhint %}

## Demo

[Example implementation using Ecotone Lite.](https://github.com/ecotoneframework/quickstart-examples/tree/master/StatefulProjection)
