---
description: Query CQRS PHP
---

# Query Handling

Be sure to read [CQRS Introduction](../) before diving in this chapter.

## Handling Queries

`External Query Handlers` are Services available in your dependency container, which are defined to handle `Queries`.

```php
class TicketService
{
    #[QueryHandler] 
    public function getTicket(GetTicketById $query) : array
    {
        //return ticket
    }
}
```

Queries are Plain Old PHP Objects:

```php
class readonly GetTicketById
{
    public function __construct(
        public string $ticketId
    ) {}
}
```

To send an Query we will be using `send` method on `QueryBus`. \
Query will be delivered to corresponding Query Handler.

{% tabs %}
{% tab title="Symfony / Laravel" %}
```php
class TicketController
{
   // Query Bus will be auto registered in Depedency Container.
   public function __construct(private QueryBus $queryBus) {}
   
   public function createTicketAction(Request $request) : Response
   {
      $result = $this->queryBus->send(
         new GetTicketById(
            $request->get("ticketId")            
         )
      );
      
      return new Response(\json_encode($result));
   }
}
```
{% endtab %}

{% tab title="Lite" %}
```php
$ticket = $messagingSystem->getQueryBus()->send(
  new GetTicketById(
    $ticketId            
  )
);
```
{% endtab %}
{% endtabs %}

## Sending with Routing

Just like with Commands, we may use routing in order to execute queries:

```php
class TicketService
{
    #[QueryHandler("ticket.getById")] 
    public function getTicket(string $ticketId) : array
    {
        //return ticket
    }
}
```

To send an Query we will be using `sendWithRouting` method on `QueryBus`. \
Query will be delivered to corresponding Query Handler.

{% tabs %}
{% tab title="Symfony / Laravel" %}
```php
class TicketController
{
   public function __construct(private QueryBus $queryBus) {}
   
   public function createTicketAction(Request $request) : Response
   {
      $result = $this->queryBus->sendWithRouting(
         "ticket.getById",
         $request->get("ticketId")            
      );
      
      return new Response(\json_encode($result));
   }
}
```
{% endtab %}

{% tab title="Lite" %}
```php
$ticket = $messagingSystem->getQueryBus()->sendWithRouting(
   "ticket.getById",
   $ticketId            
);
```
{% endtab %}
{% endtabs %}

## Converting result from Query Handler

If you have registered [Converter](../../../messaging/conversion/) for specific Media Type, then you can tell `Ecotone` to convert result of your `Query Bus` to specific format.  \
In order to do this, we need to make use of `Metadata`and `replyContentType` header.

{% tabs %}
{% tab title="Symfony / Laravel" %}
```php
class TicketController
{
   public function __construct(private QueryBus $queryBus) {}
   
   public function createTicketAction(Request $request) : Response
   {
      $result = $this->queryBus->sendWithRouting(
         "ticket.getById",
         $request->get("ticketId"),
         // Tell Ecotone which format you want in return
         expectedReturnedMediaType: "application/json"            
      );
      
      return new Response($result);
   }
}
```
{% endtab %}

{% tab title="Lite" %}
```php
$ticket = $messagingSystem->getQueryBus()->sendWithRouting(
   "ticket.getById",
   $ticketId,
   expectedReturnedMediaType: "application/json"            
);
```
{% endtab %}
{% endtabs %}

## C
