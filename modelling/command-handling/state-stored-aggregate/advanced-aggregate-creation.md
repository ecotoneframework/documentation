---
description: DDD PHP
---

# Advanced Aggregate creation

## Create an Aggregate by another Aggregate

There may be a scenario where the creation of an Aggregate is conditioned by the current state of another Aggregate.&#x20;

Ecotone provides a possibility for that and lets you focus more on domain modeling rather than technical nuances you may face trying to implement an actual use case.&#x20;

This case is supported by both Event Sourcing and State-based Aggregates.

### Create a State-based Aggregate

It is possible to send a command to an Aggregate and expect a State-based Aggregate to be returned.

```php
#[Aggregate]
final class Calendar
{
    /** @var array<string> */
    private array $meetings = [];

    public function __construct(#[Identifier] public string $calendarId) 
    {
    }

    #[CommandHandler]
    public function scheduleMeeting(ScheduleMeeting $command): Meeting
    {
        // checking business rules

        $this->meetings[] = $command->meetingId;

        return new Meeting($command->meetingId);
    }
}

#[Aggregate]
final class Meeting
{
    public function __construct(#[Identifier] public string $meetingId) 
    {
    }
}
```

### Create an Event Sourcing Aggregate

It is also possible to send a command to an Aggregate and expect the Event Sourcing Aggregate to be returned.

```php
#[Aggregate]
final class Calendar
{
    /** @var array<string> */
    private array $meetings = [];

    public function __construct(#[Identifier] public string $calendarId) 
    {
    }

    #[CommandHandler]
    public function scheduleMeeting(ScheduleMeeting $command): Meeting
    {
        // checking business rules

        $this->meetings[] = $command->meetingId;

        return Meeting::create($command->meetingId);
    }
}

#[EventSourcingAggregate(true)]
final class Meeting
{
    use WithEvents;
    use WithAggregateVersioning;
    
    #[Identifier]
    public string $meetingId;

    public static function create(string $meetingId): self
    {
        $meeting = new self();
        $meeting->recordThat(new MeetingCreated($meetingId));
        
        return $meeting;
    }
}
```

### Events handling

Both of the Aggregates (called and result) can still record their Events using an Internal Recorder. Recorded Events will be published after the operation is persisted in the database.

### Persisting a state change

In the case of an Event Sourcing Aggregate recording an event indicates a state change of that Aggregate.

Also, when calling a State-based Aggregate its state may be changed before returning the newly created Aggregate. E.g. you want to save a reference to the newly created Aggregate.

Ecotone will try to persist both called and returned Aggregates.

{% hint style="warning" %}
When splitting your aggregates into the smallest, independent parts of the domain you have to be aware of transaction boundaries which Aggregate has to protect. In the case where the creation of an Aggregate is the transaction boundary of another Aggregate, it may require a state change of the one that protects that boundary.&#x20;

This is a very specific scenario where two aggregates will persist at the same time within the same transaction which is covered by Ecotone.
{% endhint %}
