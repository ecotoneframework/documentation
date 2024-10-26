# Extending Message Buses (Gateways)

Ecotone provides ability to extend any Messaging Gateway using Interceptors. We can hook into the flow and add additional behavior.

For better understanding, please read [Interceptors section](interceptors.md) before going through this chapter.

## Intercepting Gateways

Suppose we want to add custom logging, whenever any Command is executed. \
We know that **CommandBus** is a interface for sending **Commands**, therefore we need to hook into that Gateway.

```php
class LoggerInterceptor
{
    #[Before(pointcut: CommandBus::class)]
    public function log(object $command, array $metadata) : void
    {
        // log Command message
    }
}
```

Intercepting Gateways, does not differ from intercepting Message Handlers.

## Building customized Gateways

We may also want to have different types of Message Buses for given Message Type. \
For example we could have EventBus with audit which we would use in specific cases. Therefore we want to keep the original EventBus untouched, as for other scenarios we would simply keep using it.&#x20;

To do this, we will introduce our new EventBus:

```php
interface AuditableEventBus extends EventBus {}
```

That's basically enough to register our new interface. This new Gateway will be automatically registered in our DI container, so we will be able to inject it and use.

{% hint style="success" %}
It's enough to extend given Gateway with custom interface to register new abstraction in  Gateway in Dependency Container. \
In above example **AuditableEventBus** will be automatically available in Dependency Container to use, as Ecotone will deliver implementation.
{% endhint %}

```php
#[CommandHandler]
public function placeOrder(PlaceOrder $command, AuditableEventBus $eventBus)
{
    // place order
    
    $eventBus->publish(new OrderWasPlaced());
}
```

Now as this is separate interface, we can point interceptor specifically on this

```php
class Audit
{
    #[Before(pointcut: AuditableEventBus::class)]
    public function log(object $event, array $metadata) : void
    {
        // store audit
    }
}
```

### Pointcut by attributes

We could of course intercept by attributes, if we would like to make audit functionality reusable

```php
#[Auditable]
interface AuditableEventBus extends EventBus {}
```

and then we pointcut based on the attribute

```php
class Audit
{
    #[Before(pointcut: Auditable::class)]
    public function log(object $event, array $metadata) : void
    {
        // store audit
    }
}
```

## Asynchronous Gateways

Gateways can also be extended with asynchronous functionality on which you can read more in [Asynchronous section](../asynchronous-handling/asynchronous-message-bus-gateways.md).

```php
#[Asynchronous("async")]
interface OutboxCommandBus extends CommandBus
{

}
```
