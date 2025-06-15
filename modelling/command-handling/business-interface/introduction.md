# Introduction

Be sure to read [CQRS Introduction](../) before diving in this chapter.

## Execute your Business Actions via Interface

Business Interface aims to reduce boierplate code and make your domain actions explicit. \
In Application we describe an Interface, which executes Business methods. Ecotone will deliver implementation for this interface, which will bind the interface with specific actions. \
This way we can get rid of delegation level code and focus on the things we want to achieve. \
\
For example, if we don't want to trigger action via Command/Query Bus, we can do it directly using our business interface and skip all the Middlewares that would normally trigger during Bus execution.\
There are different types of Business Interfaces and in this chapter we will discuss the basics of build our own Business Interface, in next sections we will dive into specific types of business interfaces: [Repositories](../repository/repository.md) and [Database Layers](working-with-database/).

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

This way we don't need to use Command Bus and we can bypass all Bus related interceptors.

The attribute **#\[BusinessMethod]** tells Ecotone that given **Interface** is meant to be used as entrypoint to Messaging and which Message Handler it should send the Message to. \
Ecotone will provide implementation of this interface directly in our Dependency Container.

{% hint style="success" %}
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
We may of course pass Command class if we need to pass more data to our Aggregate's Command Handler.
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

## Result Conversion

If we have registered [Converter](../../../messaging/conversion/) then we let Ecotone do conversion to **Message Handler** specific format:

```php
class TicketService
{
    #[QueryHandler("ticket.get_by_id")] 
    public function getTicket(GetTicketById $query) : array
    {
        //return ticket as array
    }
}
```

Then we may call this Query Handler using Interface

```php
interface TicketApi
{
    // return ticket as Class
    #[BusinessMethod("ticket.get_by_id")]
    public function getTicket(GetTicketById $query): TicketDTO;
}
```

Ecotone will use defined Converter to convert **`array`** to **`TicketDTO`**.

{% hint style="success" %}
Such conversion are useful in order to work with objects and to avoid writing transformation code in our business code. We can build generic queries, and transform them to different classes using different business methods.
{% endhint %}

## Payload Conversion

If we have registered [Converter](../../../messaging/conversion/) then we let Ecotone do conversion to **Message Handler** specific format:

```php
class TicketService
{
    #[CommandHandler("ticket.create")] 
    public function getTicket(CreateTicket $command) : void
    {

    }
}
```

Then we may call this Query Handler using Interface

```php
interface TicketApi
{
    #[BusinessMethod("ticket.create")]
    public function getTicket(array $query): void;
}
```

Ecotone will use defined Converter to convert **array** to **CreateTicket** command class.

{% hint style="success" %}
This type of conversion is especially useful, when we receive data from external source and we simply want to push it to given Message Handler. We avoid doing transformations ourselves, as we simply push what we receive as array.
{% endhint %}
