---
description: PHP Event Sourcing Projections
---

# Projection Introduction

Before diving into this topic, read the [Event Sourcing Introduction](../event-sourcing-introduction/) first.

## Introduction

The power of Event Sourcing is not only the full history of what happened. \
As we do have a full history, it's easy to imagine that we may want to use it for different purposes.\
And one of our purposes will be related to view this data in specific way. \


Let's take as an example our Ticket's Event Stream:

<figure><img src="../../../.gitbook/assets/ticket_event_stream_2.png" alt=""><figcaption></figcaption></figure>

In most of the situations besides knowing the history, we would also want to know how all does Tickets looks at present moment. Therefore it means we need to build an view from those events which will represent current state, and for this we use **Projections**.&#x20;

## Projections

So we may be in need to build the list of all the tickets and their current status

<figure><img src="../../../.gitbook/assets/ticket-list (1).png" alt=""><figcaption><p>List of tickets available in the system with current status</p></figcaption></figure>

For example this list can be stored in the database table with three columns:

* id
* type
* status

The difference between the traditional approach and ES approach is that we will be delivering this view from the Event Stream. Therefore this will be only the representation build up from the past events, and will be used only for reading. Data delivered from Events to shape specific view, we call **Read Models**.

## Building first Projection

Ecotone provides abstraction to quickly build new Projections, it does follow Ecotone's declarative configuration. Before we will jump into implementation, let's quickly review how our Ticket Event Sourced Aggregate could look like:

```php
#[EventSourcingAggregate]
class Ticket
{
    use WithAggregateVersioning;

    #[Identifier]
    private string $ticketId;

    public static function register(RegisterTicket $command): array
    {
        return [new TicketWasRegistered($command->id, $command->type)];
    }

    #[CommandHandler]
    public function close(CloseTicket $command) : array
    {
        return [new TicketWasClosed($this->ticketId)];
    }
}
```

So we do have two Events here, **TicketWasRegistered** and **TicketWasClosed**. \
We will be subscribing to those in order to build our new **Ticket List Projection.**\


* Let's first define our new **Ticket List Projection**

```php
#[Projection("ticket_list", Ticket::class)]
class TicketListProjection {

    // This is Connection to our Database or wherever we want to store the data
    public function __construct(private Connection $connection) {}

(...)
```

We do start by creating new class, which we mark with **Projection** attribute.\
The first argument is the name of our new projection **"ticket\_list",** the second is the related ES Aggregate **"Ticket"** from which we will be subscribing Events. We will touch on the second argument more in next sections.

* Ecotone **will take care of creating the Projection for us**, therefore we can tell it how to do it

```php
#[ProjectionInitialization]
public function initializeProjection() : void
{
    if ($this->connection->createSchemaManager()->tablesExist('ticket_list')) {
        return;
    }

    $table = new Table('ticket_list');

    $table->addColumn('id', Types::STRING);
    $table->addColumn('type', Types::STRING);
    $table->addColumn('status', Types::STRING);

    $this->connection->createSchemaManager()->createTable($table);
}
```

* Now we are ready to subscribe to Ticket related Events

```php
#[EventHandler]
public function onTicketWasPrepared(TicketWasRegistered $event) : void
{
    $this->connection->insert('ticket_list', [
        "id" => $event->id,
        "ticket_type" => $event->type,
        "status" => "open"
    ]);
}
```

This is enough for Ecotone to know that this should be triggered whenever **TicketWasRegistered** happens. As a result of triggering this Event Handler we store new ticket in the our database table

We also want to change the status, when ticket is closed, so let's add that now:

```php
#[EventHandler]
public function onTicketWasCancelled(TicketWasCancelled $event) : void
{
    $this->connection->update(
        'ticket_list, 
        ["status" => "cancelled"], 
        ["ticket_id" => $event->getTicketId()]
    );
}
```

This is all to make our Projection work. There is no any additional configuration needed as we are working from higher level abstraction. We tell what events we want to have delivered, and Ecotone will take care of delivering and initializing and triggering our Projections.

## How Projections are triggered

By default this Projection will be triggered synchronously. This means that after Event Sourced Aggregate is called, Events will first be stored in the Event Stream, and then Projection will be called.&#x20;

<figure><img src="../../../.gitbook/assets/aggregate.png" alt=""><figcaption><p>Event Sourced Aggregate stores Event in the Event Stream and then it's published</p></figcaption></figure>

Our Projection subscribe to those Events, therefore it will be triggered as a result

<figure><img src="../../../.gitbook/assets/describe (1).png" alt=""><figcaption><p>Projection is executed as a result of published Event</p></figcaption></figure>



By default projections work synchronously as part of the same process of Command Handler execution. This ensures that our Projection is always consistent with changes in the Event Stream, because it's wrapped in database transaction.&#x20;



<figure><img src="../../../.gitbook/assets/db (1) (1).png" alt=""><figcaption><p>Command Handler and Projection execution is wrapped in same transaction</p></figcaption></figure>

{% hint style="success" %}
Synchronous projects are done within same transaction as the Command execution. This way Ecotone ensures consistency of the data by default. This behaviour is configurable.
{% endhint %}

This works well for scenarios when there are no much changes to happening to given instance of Event Sourced Aggregate. How Ecotone handles Concurrency was described in more details in [previous section](../event-sourcing-introduction/working-with-event-streams.md). \
What is important is that for low writes this solution will work perfectly, for high volume of writes on other hand we may want to trigger Projections asynchronously.

{% hint style="success" %}
Synchronous projections are simpler in development, as we can immediately fetch the data from Read Model and be sure that is consistent with the changes.\
With Asynchronous data may be refreshed after some time, therefore if fetched immediately, we may get stale results.
{% endhint %}

## Asynchronous Projections

Ecotone provides great abstraction for making the code asynchronous. From development perspective the code stays the same like synchronous, yet under the hood thanks to Messaging abstraction it can be easily switched to work [asynchronously via Message Channels](../../asynchronous-handling/asynchronous-message-handlers.md).&#x20;

So to make the Projection asynchronous, the only thing which we need to do, is to mark it as asynchronous.

```php
#[Asynchronous('async')]
#[Projection("ticket_list", Ticket::class)]
class TicketListProjection
```

Ecotone will take care of delivering the triggering Event via given **async** channel to the Projection. \
This way we can start with synchronous projections, and when we will feel the need, simply switch them to Asynchronous without any single line of code being changed. \
You may read more about execution process in [next section](executing-and-managing/).\
\
We need to touch on one more important topic. Where do we actually get the data from for Projections.

## The source of data for Projection

Events that trigger Projections are not actually a source of the data for them.\
This is because if we would lose Event Message along the way due some failure (For example we don't use [Outbox](../../recovering-tracing-and-monitoring/resiliency/outbox-pattern.md)) or it would and in [Dead Letter](../../recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter.md) then we would basically skip over an Event.

Let's take as an example Asynchronous Projection, where we want to store Ticket with new "**alert-warning**" type. However let's suppose we've created column with limited size for type - which is up to  10 characters. Therefore our Projection will fail on storing that, because the type is 13 characters long:

<figure><img src="../../../.gitbook/assets/alert-warning.png" alt=""><figcaption></figcaption></figure>

Now if after that we will receive **Ticket was closed** event, then related Event Handler would update nothing, as there is no this Ticket stored in our Read Model:

<figure><img src="../../../.gitbook/assets/closed-2.png" alt=""><figcaption></figcaption></figure>

So this is obviously not way of ensuring consistency in the system. \
Ecotone does it differently and treats the incoming Events just as "triggers". \
It works like information for the Projection to fetch the Events from the Event Stream and start Projecting.\


<figure><img src="../../../.gitbook/assets/projection (2).png" alt=""><figcaption><p>Projection fetches the Event from Event Stream. Incoming Event is just a trigger.</p></figcaption></figure>

This way if even so Ticket Was Registered failed, when Ticket was Closed would come after it would still get the original event first. So if we would fix the problem with column size, it would basically self-heal automatically.&#x20;

{% hint style="success" %}
Each Projection keep track of it's position. Therefore whenever new Event comes in, it knows from which point in Event Stream it should fetch the Events from.
{% endhint %}

There is one more important reason for building Projections from Event Stream, **Projection Rebuilding**.

### Setting up new Projections and Rebuilding

Thanks to Projections ability to build the projection from the Event Stream, we are not bound by time. When we deploy new Projection, it will go over previous Events as part of the projecting process.\
This way we can ability to be able to ship new Projections at any point of time, yet with ability to use all the previous Events from the past.&#x20;

Besides that we can rebuild existing Projection, as rebuilding is all about reseting the Projection's position, and start to fetch from scretch. You may read about available actions in [Executing and Managing section](executing-and-managing/).
