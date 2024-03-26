---
description: DDD PHP
---

# Aggregate Event Handlers

Read [Aggregate Introduction](./) sections first to get more details about Aggregates.

## Publishing Events from Aggregate

To tell `Ecotone` to retrieve Events from your Aggregate add trait `WithEvents` which contains two methods: `recordThat` and `getRecordedEvents`.

{% hint style="info" %}
As Ecotone never forces to use framework specific classes in your business code, you may replace it with your own implementation.
{% endhint %}

After importing trait, Events will be automatically retrieved and published after handling Command in your Aggregate.

```php
#[Aggregate]
class Ticket
{
    // Import trait with recordThat method
    use WithEvents;

    #[Identifier]
    private Uuid $ticketId;
    private string $description;
    private string $assignedTo;
       
    #[CommandHandler]
    public function changeTicket(ChangeTicket $command): void
    {
        $this->description = $command->description;
        $this->assignedTo = $command->assignedTo;
        
        // Record the event
        $this->recordThat(new TicketWasChanged($this->ticketId));
    }
}
```

{% hint style="success" %}
Using `recordThat` will delay sending an event till the moment your Aggregate is saved in the Repository. This way you ensure that no Event Handlers will be called before the state is actually stored.
{% endhint %}

## Subscribing to Event from your Aggregate

Sometimes you may have situation, where Event from one Aggregate will actually change another Aggregate. In those situations you may actually subscribe to the Event directly from Aggregate, to avoid creating higher level boilerplate code.

```php
#[Aggregate]
class Promotion
{
    #[Identifier]
    private Uuid $userId;
    private bool $isActive;
       
    #[EventHandler]
    public function stop(AccountWasClosed $event): void
    {
        $this->isActive = false;
    }
}
```

In those situations however you need to ensure event contains of reference id, so Ecotone knows which Aggregate to load from the database.

```php
class readonly AccountWasClosed
{
    public function __construct(public Uuid $userId) {}
}
```

{% hint style="success" %}
For more sophisticated scenarios, where there is no direct identifier in corresponding event, you may use of identifier mapping. You can read about it more in [Saga related section.](../saga.md#targeting-identifier-from-event-command)
{% endhint %}

## Sending Named Events

You may subscribe to Events by names, instead of the class itself. This is useful in cases where we want to decoupled the modules more, or we are not interested with the Event Payload at all. \
\
For Events published from your Aggregate, it's enough to provide `NamedEvent` attribute with the name of your event.

```php
#[NamedEvent('order.placed')]
final readonly class OrderWasPlaced
{
    public function __construct(
        public string $orderId,
        public string $productId
    ) {}
}
```

And then you can subscribe to the Event using name

```php
#[EventHandler(listenTo: "order.placed")]
public function notify(#[Header("executoId")] $executorId): void
{
    // notify user that the Order was placed
}
```
