---
description: Commands CQRS PHP
---

# CQRS Introduction - Commands

In this section we will discuss using Commands, Events and Queries. \
We will start with Commands and Command Handlers. Even so we will be discussing Commands the functionality which we will tackle, applies to `Queries` and `Events` also. \
Understanding this part will give us understanding of the whole.

## Handling Commands

`External Command Handlers` are Services available in Dependency Container, which are defined to handle `Commands`. We call them External to differentiate from Aggregate Command Handlers, which will be described in later part of the section.\
\
In Ecotone we enable Command Handlers using attributes. By marking given method with               #`[CommandHandler]` we state it should be used as a `Command Handler`.

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

{% hint style="success" %}
In case of Ecotone the class itself is not a Command Handler, it's a method that is considered to be Command Handler. This way we may join multiple Command Handlers under same class without introducing new classes if that's not needed.
{% endhint %}

`Command Handlers` are methods where we would typically places our business logic.\
In above example using `#[CommandHandler]` we stated that `createTicket` method will be handling `CreateTicketCommand`. \
The first parameter of Command Handler method is indicator of the Command Class we want to handle, so in this case it will be `CreateTicketCommand`.\
\
Now whenever we will send this `CreateTicketCommand` using `Command Bus`, it will be delivered to `createTicket` method.

{% hint style="info" %}
If you are using autowire functionality, then all your classes are registered using class names. \
In other case, if your class name is not corresponding to their name in Dependency Container, we may use `ClassReference:`

```php
#[ClassReference("ticketService")]
class TicketService
```
{% endhint %}

## Sending Commands

We send Command using `Command Bus.` After Ecotone is installed all Buses are available out of the box in Dependency Container, this way we may start using them directly after installation.\
\
In order to use the Command we first need to define it:

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
All Messages (Command/Queries/Events) just like Message Handlers (Command/Query/Event Handlers) are simple Plain old PHP Objects, which means they do not extend or implement any framework specific classes. \
This way we keep our business code clean and easy to understand.
{% endhint %}

To send an Command we will be using `send` method on `CommandBus`. \
Command will be delivered to corresponding Command Handler.

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

We may send Command with `Metadata` (Message Headers) via Command Bus.\
This way we may provide additional information that should not be part of the Command or details which will be reused across `Command Handlers` without copy/pasting it to each of the related `Command` classes.

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

`#[Header]` provides information that we would like to fetch metadata which is under `executorId`. This way Ecotone knows what to pass to the Command Handler.

{% hint style="success" %}
If we use [Asynchronous](../../asynchronous-handling/) Command Handler, Ecotone will ensure our metadata will be serialized and deserialized correctly.
{% endhint %}

## Injecting Services  into Command Handler

If we need additional Services (which are available in Dependency Container) to perform our business logic, we may pass them to the Command Handler using `#[Reference]` attribute:

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

In Ecotone we may register Command Handlers under routing instead of a class name. \
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
Ecotone is using message routing for [cross application communication](../../microservices-php/distributed-bus.md). This way applications can stay decoupled from each other, as there is no need to share the classes between them.
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

It happens that after performing action, we would like to return some value. \
This may happen for scenarios that require immediate response, for taking an payment may generate redirect URL for the end user.&#x20;

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
Take under consideration that returning works for synchronous Command Handlers. \
in case of asynchronous scenarios this will not be possible.
{% endhint %}

[^1]: 
