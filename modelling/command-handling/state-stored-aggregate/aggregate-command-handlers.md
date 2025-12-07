---
description: DDD PHP
---

# Aggregate Command Handlers

Read [Aggregate Introduction](./) sections first to get more details about Aggregates.

## Aggregate Factory Method

New Aggregates are initialized using public factory method (static method).

```php
#[Aggregate]
class Ticket
{
    #[Identifier]
    private Uuid $ticketId;
    private string $description;
    private string $assignedTo;
    
    #[CommandHandler]
    public static function createTicket(CreateTicket $command): static
    {
        $ticket = new self();
        $ticket->id = Uuid::generate();
        $ticket->assignedTo = $command->assignedTo;
        $ticket->description = $command->description;

        return $ticket;
    }
}
```

After calling `createTicket` aggregate will be automatically stored.

{% hint style="success" %}
Factory method is static method in the Aggregate class.\
You may have multiple factory methods if needed.
{% endhint %}

Sending Command looks exactly the same like in [External Command Handlers](../external-command-handlers/) scenario.

```php
$ticketId = $this->commandBus->send(
   new CreateTicket($assignedTo, $description)
);
```

{% hint style="success" %}
When factory method is called from Command Bus, then Ecotone will return new assigned identifier.
{% endhint %}

## Aggregate Action Method

Aggregate actions are defined using public method (non-static). Ecotone will ensure loading and saving the aggregate after calling action method.

```php
#[Aggregate]
class Ticket
{
    #[Identifier]
    private Uuid $ticketId;
    private string $description;
    private string $assignedTo;
       
    #[CommandHandler]
    public function changeTicket(ChangeTicket $command): void
    {
        $this->description = $command->description;
        $this->assignedTo = $command->assignedTo;
    }
}
```

`ChangeTicket` should contain the identifier of Aggregate instance on which action method should be called.

```php
class readonly ChangeTicket
{
    public function __construct(
        public Uuid $ticketId;
        public string $description;
        public string $assignedTo
    ) {}
}
```

And then we call it from `Command Bus`:

```php
$ticketId = $this->commandBus->send(
   new ChangeTicket($ticketId, $description, $assignedTo)
);
```

## Calling Aggregate without Command Class

In fact we don't need to provide identifier in our Commands in order to execute specific Aggregate instance. We may not need a Command class in specific scenarios at all.

```php
#[Aggregate]
class Ticket
{
    #[Identifier]
    private Uuid $ticketId;
    private bool $isClosed = false;
       
    #[CommandHandler("ticket.close")]
    public function close(): void
    {
        $this->isClosed = true;
    }
}
```

In this scenario, if we would add `Command Class`, it would only contain of the identifier and that would be unnecessary boilerplate code. To solve this we may use [Metadata](../external-command-handlers/#sending-commands-with-metadata) in order to provide information about instance of the aggregate we want to call.

```php
$this->commandBus->sendWithRouting(
    "ticket.close", 
    metadata: ["aggregate.id" => $ticketId]
)
```

`"aggregate.id"` is special metadata that provides information which aggregate we want to call.

{% hint style="success" %}
When we avoid creating Command Classes with identifiers only, we decrease amount of boilerplate code that we need to maintain.
{% endhint %}

## Redirected Aggregate Creation

There may be a cases where you would like to do conditional logic, if aggregate exists do thing, otherwise this. This may be useful to keep our higher level code clean of "if" statements and to simply API by exposing single method.

```php
#[Aggregate]
class Ticket
{
    #[Identifier]
    private Uuid $ticketId;
    private string $description;
    private string $assignedTo;
    
    #[CommandHandler]
    public static function createTicket(CreateTicket $command): static
    {
        $ticket = new self();
        $ticket->id = Uuid::generate();
        $ticket->assignedTo = $command->assignedTo;
        $ticket->description = $command->description;

        return $ticket;
    }
    
    #[CommandHandler]
    public function changeTicket(CreateTicket $command): void
    {
        $this->description = $command->description;
        $this->assignedTo = $command->assignedTo;
    }
}
```

Both Command Handlers are registered for same command `CreateTicket`, yet one method is `factory method` and the second is `action method`.\
When Command will be sent, Ecotone will try to load the aggregate first,\
if it will be found then `changeTicket` method will be called, otherwise `createTicket`.

{% hint style="success" %}
Redirected aggregate creation works the same for Event Sourced Aggregates.
{% endhint %}

## Publishing Events from Aggregate

For standard Aggregates (non Event-Sourced) we can use **WithEvents trait** or provide method that with **AggregateEvents attribute** to provide list of Events that Aggregate has recorded.\
After saving changes to the Aggregate, Ecotone will automatically publish related Events

```php
#[Aggregate]
class Ticket
{
    use WithEvents; // Provides methods for collecting events

    #[Identifier]
    private Uuid $ticketId;
    
    #[CommandHandler]
    public static function createTicket(
        CreateTicket $command
    ): static
    {
        $self = new self($command->id);
        
        $self->recordThat(new TicketWasCreated($command->id));
        
        return $self;
    }
}
```

## Calling Aggregate with additional arguments

Just as standard Command Handler, we can pass Metadata and DI Services to our Aggregates.&#x20;

```php
#[Aggregate]
class Ticket
{
    #[Identifier]
    private Uuid $ticketId;
    
    #[CommandHandler]
    public static function createTicket(
        CreateTicket $command,
        #[Header("executorId")] string $executorId,
        #[Reference] Clock $clock,
    ): static
    {
        return new self(
            $command->id,
            $executorId,
            $clock->currentTime(),
        );
    }
}
```

