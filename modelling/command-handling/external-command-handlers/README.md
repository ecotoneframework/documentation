---
description: Commands CQRS PHP
---

# CQRS Introduction - Commands

**In this section, we will look at how to use Commands, Events, and Queries.**\
This will help you understand the basics of Ecotone’s CQRS support and how to build a message-driven application.

Command Handlers are methods where we typically place our business logic, so we’ll start by exploring how to use them.

## Handling Commands

**Any service available in your Dependency Container can become a Command Handler.**\
Command Handlers are responsible for performing business actions in your system.\
In Ecotone-based applications, you register a Command Handler by adding the `CommandHandler` attribute to the specific method that should handle the command:

```php
class TicketService
{
    #[CommandHandler] 
    public function createTicket(CreateTicketCommand $command) : void
    {
        // handle create ticket command
    }
}
```

In the example above, the **#\[CommandHandler]** attribute tells Ecotone that the "createTicket" method should handle the **CreateTicketCommand**.

The first parameter of a Command Handler method determines which command type it handles — in this case, it is `CreateTicketCommand`.

{% hint style="success" %}
**In Ecotone, the class itself is not a Command Handler — only the specific method is.**\
This means you can place multiple Command Handlers inside the same class, to make correlated actions available under same API class.
{% endhint %}

{% hint style="info" %}
**If you are using autowiring, all your classes are registered in the container under their class names.**\
This means Ecotone can automatically resolve them without any extra configuration.

If your service is registered under a different name in the Dependency Container, you can use `ClassReference` to point Ecotone to the correct service:

```php
#[ClassReference("ticketService")]
class TicketService
```
{% endhint %}

## Sending Commands

We send a Command using the Command Bus. After installing Ecotone, all Buses are automatically available in the Dependency Container, so we can start using them right away.\
\
Before we can send a Command, we first need to define it:

```php
class readonly CreateTicketCommand
{
    public function __construct(
        public string $priority,
        public string $description
    ){}
}
```

{% hint style="success" %}
**All Messages (Commands, Queries, and Events), as well as Message Handlers, are just plain PHP objects.**\
They don’t need to extend or implement any Ecotone-specific classes.\
This keeps your business code clean, simple, and easy to understand.
{% endhint %}

To send a command, we use the `send` method on the `CommandBus`. \
The command gets automatically routed to its corresponding Command Handler

{% tabs %}
{% tab title="Symfony / Laravel" %}
```php
class TicketController
{
   // Command Bus will be auto registered in Depedency Container.
   public function __construct(private CommandBus $commandBus) {}
   
   public function createTicketAction(Request $request) : Response
   {
      $this->commandBus->send(
         new CreateTicketCommand(
            $request->get("priority"),
            $request->get("description"),            
         )
      );
      
      return new Response();
   }
}
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->getCommandBus()->send(
    new CreateTicketCommand(
        $priority,
        $description,            
     )
);
```
{% endtab %}
{% endtabs %}

## Sending Commands with Metadata

We can send commands with **metadata (also called Message Headers)** through the Command Bus. \
This lets us include **additional context that doesn't belong in the command itself**, or share information across multiple Command Handlers without duplicating it in each command class.

{% tabs %}
{% tab title="Symfony / Laravel" %}
```php
class TicketController
{
   public function __construct(private CommandBus $commandBus) {}
   
   public function closeTicketAction(Request $request, Security $security) : Response
   {
      $this->commandBus->send(
         new CloseTicketCommand($request->get("ticketId")),
         ["executorId" => $security->getUser()->getId()]
      );
   }
}
```
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

And then to access given metadata, we will be using Header attribute:

{% tabs %}
{% tab title="Command Handler" %}
```php
class TicketService
{   
    #[CommandHandler]
    public function closeTicket(
        CloseTicketCommand $command, 
        // by adding Header attribute we state what metadata we want to fetch
        #[Header("executorId")] string $executorId
    ): void
    {          
//        handle closing ticket with executor from metadata
    }   
}
```
{% endtab %}
{% endtabs %}

The `#[Header]` attribute tells Ecotone to fetch a specific piece of metadata using the key `executorId`. This way, Ecotone knows exactly which metadata value to pass into our Command Handler.

{% hint style="success" %}
If we use [Asynchronous](../../asynchronous-handling/) Command Handler, Ecotone will ensure our metadata will be serialized and deserialized correctly.
{% endhint %}

## Injecting Services into Command Handler

If we need additional services from the Dependency Container to handle our business logic, we can inject them into our Command Handler using the `#[Reference]` attribute:

```php
class TicketService
{   
    #[CommandHandler]
    public function closeTicket(
        CloseTicketCommand $command, 
        #[Reference] AuthorizationService $authorizationService
    ): void
    {          
//        handle closing ticket with executor from metadata
    }   
}
```

In case Service is defined under custom id in DI, we may pass the reference name to the attribute:

```php
#[Reference("authorizationService")] AuthorizationService $authorizationService
```

## Sending Commands via Routing

In Ecotone we may register Command Handlers under routing instead of a class name.\
This is especially useful if we will register [Converters](../../../messaging/conversion/) to tell Ecotone how to deserialize given Command. This way we may simplify higher level code like `Controllers` or `Console Line Commands` by avoid transformation logic.

{% tabs %}
{% tab title="Symfony / Laravel" %}
```php
class TicketController
{
   public function __construct(private CommandBus $commandBus) {}
   
   public function createTicketAction(Request $request) : Response
   {
      $commandBus->sendWithRouting(
         "createTicket", 
         $request->getContent(),
         "application/json" // we tell what format is used in the request content
      );
      
      return new Response();
   }
}
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->getCommandBus()->sendWithRouting(
   "createTicket", 
   $data,
   "application/json"
);
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="Command Handler" %}
```php
class TicketService
{   
    // Ecotone will do deserialization for the Command
    #[CommandHandler("createTicket")]
    public function createTicket(CreateTicketCommand $command): void
    {
//        handle creating ticket
    }   
}
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Ecotone is using message routing for [cross application communication](../../microservices-php/distributed-bus/). This way applications can stay decoupled from each other, as there is no need to share the classes between them.
{% endhint %}

## Routing without Command Classes

There may be cases where creating Command classes is unnecessary boilerplate, in those situations, we may simplify the code and make use `scalars`, `arrays` or `non-command classes` directly.

{% tabs %}
{% tab title="Symfony / Laravel" %}
```php
class TicketController
{
   private CommandBus $commandBus;

   public function __construct(CommandBus $commandBus)
   {
       $this->commandBus = $commandBus;   
   }
   
   public function closeTicketAction(Request $request) : Response
   {
      $commandBus->sendWithRouting(
         "closeTicket", 
         Uuid::fromString($request->get("ticketId"))
      );
      
      return new Response();
   }
}
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->getCommandBus()->sendWithRouting(
   "closeTicket", 
   Uuid::fromString($ticketId)
);
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="Command Handler" %}
```php
class TicketService
{   
    #[CommandHandler("closeTicket")]
    public function closeTicket(UuidInterface $ticketId): void
    {
//        handle closing ticket
    }   
}
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Ecotone provides flexibility which allows to create Command classes when there are actually needed. In other cases we may use routing functionality together with simple types in order to fulfill our business logic.
{% endhint %}

## Returning Data from Command Handler

It happens that after performing action, we would like to return some value.\
This may happen for scenarios that require immediate response, for taking an payment may generate redirect URL for the end user.

```php
class PaymentService
{   
    #[CommandHandler]
    public function closeTicket(MakePayment $command): Url
    {
//        handle making payment

        return $paymentUrl;
    }   
}
```

The returned data will be available as result of the Command Bus.

```php
$redirectUrl = $this->commandBus->send($command);
```

{% hint style="success" %}
Take under consideration that returning works for synchronous Command Handlers.\
in case of asynchronous scenarios this will not be possible.
{% endhint %}
