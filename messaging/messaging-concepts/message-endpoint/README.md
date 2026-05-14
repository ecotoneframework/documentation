---
description: Message Endpoint PHP
---

# Message Endpoints/Handlers

![](<../../../.gitbook/assets/endpoint (1).jpg>)

Every piece of code that processes a message in Ecotone is an **Endpoint**. `Command Handlers`, `Event Handlers`, `Query Handlers` are the bus-facing flavors — wired to the Command/Event/Query buses. **Internal Handlers**, **Splitters**, and **Routers** are the lower-level flavors — they sit on a named channel and aren't reachable from a bus.

Reach for the lower-level family when:

- You're building a step inside an Orchestrator that shouldn't be invokable as a Command from outside.
- You're processing the result of an `outputChannelName` chain.
- You need a router or splitter inside a workflow.

If your code is the entry point for an HTTP request or a public domain action, use a Command/Event/Query Handler. If it's a step in a pipeline, use an Internal Handler.

You don't implement endpoints directly or build messages by hand — you write plain PHP objects, and Ecotone connects your domain code to the messaging infrastructure through declarative configuration.&#x20;



All Message Handlers follows simple low level interface

```php
interface MessageHandler
{
    /**
     * Handles given message
     */
    public function handle(Message $message): void;
}
```

{% hint style="info" %}
`Command Handlers, Event Handlers, Query Handlers` are also Endpoints.
{% endhint %}
