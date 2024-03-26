# Business Interface

Be sure to read [CQRS Introduction](../) before diving in this chapter.

## Execute your Business Actions via Interface

Business Interface aims is to reduce boierplate code and make your domain actions explicit. \
We provide an Interface with methods and Ecotone delivers implementation. This way we can get rid of delegation level code and focus on the things we want to achieve. \
For example, if we don't want to trigger action via Command/Query Bus, we can do it directly using our business interface and skip all the Middlewares that would normally trigger during Bus execution.

There are different types of Business Interfaces and in this chapter we will discuss the basics and in another two, we will dive into abstracting away [Repositories](repository.md) and [Database Layers](working-with-database/).

## Command Interface

Let's take as an example creating new Ticket

```php
class TicketService
{
    #[CommandHandler("ticket.create")] 
    public function createTicket(CreateTicketCommand $command) : void
    {
        // handle create ticket command
    }
}
```

We may define interface, that will call this Command Handler whenever will be executed.

```php
interface TicketApi
{
    #[BusinessMethod('ticket.create')]
    public function create(CreateTicketCommand $command): void;
}
```

This way we don't need to use Command Bus and can bypass all Bus related interceptors.

The attribute `#[BusinessMethod]` tells Ecotone that given `interface` is meant to be used as entrypoint to Messaging and which Message Handler it should send the Message to. \
Ecotone will provide implementation of this interface directly in your Dependency Container.

{% hint style="success" %}
We already used Business Interfaces without being aware, `Command`, `Query` and `Event` Buses are Business Interfaces.\
From lower level API `Business Method` is actually a [Message Gateway](../../../messaging/messaging-concepts/messaging-gateway.md).
{% endhint %}

## Aggregate Command Interface

We may also execute given Aggregate directly using Business Interface.

```php
#[Aggregate]
class Ticket
{
    #[Identifier]
    private Uuid $ticketId;
    private bool $isClosed;
       
    #[CommandHandler("ticket.close")]
    public function close(): void
    {
        $this->isClosed = true;
    }
}
```

Then we define interface:

```php
interface TicketApi
{
    #[BusinessMethod('ticket.close')]
    public function create(#[Identifier] Uuid $ticketId): void;
}
```

{% hint style="success" %}
We may of course pass Command class, if we need to pass data to our Aggregate's Command Handler.
{% endhint %}

## Query Interface

Defining Query Interface works exactly the same as Command Interface and we may also use it with Aggregates.

```php
class TicketService
{
    #[QueryHandler("ticket.get_by_id")] 
    public function getTicket(GetTicketById $query) : array
    {
        //return ticket
    }
}
```

Then we may call this Query Handler using Interface

```php
interface TicketApi
{
    #[BusinessMethod("ticket.get_by_id")]
    public function getTicket(GetTicketById $query): array;
}
```

## Query Interface Conversion

If we have registered [Converter](../../../messaging/conversion/) then we let`Ecotone` convert the result of your `Query Handler` to specific format. &#x20;

```php
class TicketService
{
    #[QueryHandler("ticket.get_by_id")] 
    public function getTicket(GetTicketById $query) : array
    {
        //return ticket
    }
}
```

Then we may call this Query Handler using Interface

```php
interface TicketApi
{
    #[BusinessMethod("ticket.get_by_id")]
    public function getTicket(GetTicketById $query): TicketDTO;
}
```

Ecotone will use defined Converter to conver `array` to `TicketDTO`.

{% hint style="success" %}
Such conversion are useful in order to work with objects and to avoid writing transformation code in our business code.
{% endhint %}
