# Applying Events

As [mentioned earlier](../), Events are stored in form of a Event Stream.\
Event Stream is audit of Events, which happened in the past. \
However to protect our business invariants, we may want to work with current state of the Aggregate to know, if given action is possible or not (business invariants).&#x20;

## Business Invariants

Business Invariants in short are our simple "**if"** statements inside the Command Handler in the Aggregate. Those protect our Aggregate from moving into incorrect state.  \
With State-Stored Aggregates, we always have current state of the Aggregate, therefore we can check the invariants right away. \
With Event-Sourcing Aggregates, we store them in form of an Events, therefore we need to rebuild our Aggregate, in order to protect the invariants.



Suppose we have Ticket Event Sourcing Aggregate.

```php
#[EventSourcingAggregate]
class Ticket
{
    use WithAggregateVersioning;

    #[Identifier]
    private string $ticketId;

    (...)

    #[CommandHandler]
    public function assign(AssignPerson $command) : array
    {
        return [new PersonWasAssigned($this->ticketId, $command->personId)];
    }
}
```

For this Ticket we do allow for assigning an Person to handle the Ticket. \
Let's suppose however, that Business asked us to allow only one Person to be assigned to the Ticket at time. With current code we could assign unlimited people to the Ticket, therefore we need to protect this invariant.&#x20;

To check if whatever Ticket was already assigned to a Person, our Aggregate need to have state applied which will tell him whatever the Ticket was already assigned.\
To do this we use **EventSourcingHandler attribute passing as first argument given Event class.** This method will be called on reconstruction of this Aggregate. So when this Aggregate will be loaded, if given Event was recorded in the Event Stream, method will be called:

```php
#[EventSourcingAggregate]
class Ticket
{
    use WithAggregateVersioning;

    #[Identifier]
    private string $ticketId;
    private bool $isAssigned;

    #[CommandHandler]
    public function assign(AssignPerson $command) : array
    {
        if ($this-isAssigned) {
           throw new \InvalidArgumentException("Ticket already assigned");
        }
    
        return [new PersonWasAssigned($this->ticketId, $command->personId)];
    }

    #[EventSourcingHandler]
    public function applyPersonWasAssigned(PersonWasAssigned $event) : void
    {
        $this->isAssigned = true;
    }
}
```

Then this state, can be used in the Command Handler to decide whatever we can trigger an action or not:

```php
if ($this-isAssigned) {
  throw new \InvalidArgumentException("Ticket already assigned");
}
```

{% hint style="success" %}
As you can see, it make sense to only assign to the state attributes that protect our invariants. This way the Aggregate stays readable and clean of unused information.
{% endhint %}
