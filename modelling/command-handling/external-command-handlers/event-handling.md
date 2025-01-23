---
description: Event CQRS PHP
---

# Event Handling

Be sure to read [CQRS Introduction](../) before diving in this chapter.

## Handling Events

`External Event Handlers` are Services available in your dependency container, which are defined to handle `Events`.

```php
class TicketService
{
    #[EventHandler] 
    public function when(TicketWasCreated $event): void
    {
        // handle event
    }
}
```

Events are Plain Old PHP Objects:

```php
class readonly TicketWasCreated
{
    public function __construct(
        public string $ticketId
    ) {}
}
```

{% hint style="success" %}
The difference between Events and Command is in intention. Commands are meant to trigger an given action and events are information that given action was performed successfully.&#x20;
{% endhint %}

```php
class TicketService
{
    #[CommandHandler] 
    public function createTicket(
        CreateTicketCommand $command,
        EventBus $eventBus
    ) : void
    {
        // handle create ticket command
        
        $eventBus->publish(new TicketWasCreated($ticketId));
    }
}
```

`EventBus` is available in your Dependency Container by default, just like Command and Query buses. You may use Ecotone's feature to inject it directly into your Command Handler's method.&#x20;

{% hint style="success" %}
Just like EventBus is injected directly into your Command Handler, you may inject any other Service. This way you may make is clear what object your Command Handler needs in order to perform his action.
{% endhint %}

## Multiple Subscriptions

Unlike Command Handlers which points to specific Command Handler, Event Handlers can have multiple subscribing Event Handlers.

```php
class TicketService
{
    #[EventHandler] 
    public function when(TicketWasCreated $event): void
    {
        // handle event
    }
}

class NotificationService
{
    #[EventHandler] 
    public function when(TicketWasCreated $event): void
    {
        // handle event
    }
}
```

{% hint style="success" %}
Each Event Handler can be defined as [Asynchronous](../../asynchronous-handling/). If multiple Event Handlers are marked for asynchronous processing, each of them is handled in isolation. This ensures that in case of failure, we can safely retry, as only failed Event Handler will be performed again.
{% endhint %}

## Subscribe to Interface or Abstract Class

If your Event Handler is interested in all Events around specific business concept, you may subscribe to Interface or Abstract Class.

```php
interface TicketEvent
{
}
```

```php
class readonly TicketWasCreated implements TicketEvent
{
    public function __construct(
        public string $ticketId
    ) {}
}

class readonly TicketWasCancelled implements TicketEvent
{
    public function __construct(
        public string $ticketId
    ) {}
}
```

And then instead of subscribing to `TicketWasCreated` or `TicketWasCancelled`, we will subscribe to `TicketEvent`.

```php
#[EventHandler]
public function notify(TicketEvent $event) : void
{
   // do something with $event
}
```

## Subscribing by Union Classes

We can also subscribe to different Events using union type hint. This way we can ensure that only given set of events will be delivered to our Event Handler.

```php
#[EventHandler]
public function notify(TicketWasCreated|TicketWasCancelled $event) : void
{
   // do something with $event
}
```

## Subscribing to All Events

We may subscribe to all Events published within the application. To do it we type hint for generic `object`.

```php
#[EventHandler]
public function log(object $event) : void
{
   // do something with $event
}
```

## Subscribing to Events by Routing

Events can also be subscribed by Routing.

```php
class TicketService
{
    #[EventHandler("ticket.was_created")] 
    public function when(TicketWasCreated $event): void
    {
        // handle event
    }
}
```

And then Event is published with routing key

```php
class TicketService
{
    #[CommandHandler] 
    public function createTicket(
        CreateTicketCommand $command,
        EventBus $eventBus
    ) : void
    {
        // handle create ticket command
        
        $eventBus->publishWithRouting(
            "ticket.was_created",
            new TicketWasCreated($ticketId)
        );
    }
}
```

{% hint style="success" %}
Ecotone is using message routing for [cross application communication](../../microservices-php/distributed-bus/). This way applications can stay decoupled from each other, as there is no need to share the classes between them.
{% endhint %}

## Sending Events with Metadata

Just like with `Command Bus`, we may pass metadata to the `Event Bus`:

```php
class TicketService
{
    #[CommandHandler] 
    public function createTicket(
        CreateTicketCommand $command,
        EventBus $eventBus
    ) : void
    {
        // handle create ticket command
        
        $eventBus->publish(
            new TicketWasCreated($ticketId),
            metadata: [
                "executorId" => $command->executorId()
            ]
        );
    }
}
```

```php
class TicketService
{
    #[EventHandler] 
    public function when(
        TicketWasCreated $event,
        // access metadata with given name
        #[Header("executorId")] string $executorId
    ): void
    {
        // handle event
    }
}
```

{% hint style="success" %}
If you make your Event Handler [Asynchronous](../../asynchronous-handling/), Ecotone will ensure your metadata will be serialized and deserialized correctly.
{% endhint %}

## Metadata Propagation

By default Ecotone will ensure that your Metadata is propagated. \
This way you can simplify your code by avoiding passing around Headers and access them only in places where it matters for your business logic.

To better understand that, let's consider example in which we pass the metadata to the Command.

{% tabs %}
{% tab title="Symfony / Laravel" %}
<pre class="language-php"><code class="lang-php">class TicketController
{
   public function __construct(private CommandBus $commandBus) {}
   
   public function closeTicketAction(Request $request, Security $security) : Response
   {
      $this->commandBus-><a data-footnote-ref href="#user-content-fn-1">se</a>nd(
         new CloseTicketCommand($request->get("ticketId")),
         ["executorId" => $security->getUser()->getId()]
      );
   }
}
</code></pre>
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->getCommandBus()->send(
   new CloseTicketCommand($ticketId),
   ["executorId" => $executorId]
);
```
{% endtab %}
{% endtabs %}

However in order to perform closing ticket logic, information about the `executorId` is not needed, so we don't access that.

{% tabs %}
{% tab title="Command Handler" %}
```php
class TicketService
{   
    #[CommandHandler]
    public function closeTicket(
        CloseTicketCommand $command, 
        EventBus $eventBus
    )
    {     
        // close the ticket
             
        // we simply publishing an Event, we don't pass any metadata here 
        $eventBus->publish(new TicketWasCreated($ticketId));
    }   
}
```
{% endtab %}
{% endtabs %}

However Ecotone will ensure that your metadata is propagated from Handler to Handler.\
This means that the context is preserved and you will be able to access executorId in your Event Handler.

```php
class AuditService
{
    #[EventHandler] 
    public function log(
        TicketWasCreated $event,
        // access metadata with given name
        #[Header("executorId")] string $executorId
    ): void
    {
        // handle event
    }
}
```

[^1]: 
