---
description: DDD PHP
---

# Aggregate Query Handlers

Read [Aggregate Introduction](./) sections first to get more details about Aggregates.

## Aggregate Query Action

Aggregate actions are defined using public method (non-static). Ecotone will ensure loading in order to execute the query method.

```php
#[Aggregate]
class Ticket
{
    #[Identifier]
    private Uuid $ticketId;
    private string $assignedTo;
       
    #[QueryHandler("ticket.get_assigned_person")]
    public function getAssignedTo(): string
    {
       return $this->assignedTo;
    }
}
```

And then we call it from `Query Bus`:

```php
$this->commandBus->sendWithRouting(
    "ticket.get_assigned_person",
    // We provide instance of Ticket aggregate using metadata 
    metadata: ["aggregate.id" => $ticketId]
)
```

{% hint style="success" %}
You may of course use of Query class or metadata in case of need, which will be passed to your aggregate's method.
{% endhint %}
