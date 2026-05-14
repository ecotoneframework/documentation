---
description: Service Activator PHP
---

# Internal Message Handler

You're building a workflow step that processes a message but shouldn't be invokable from the Command Bus — for example, a "validate eligibility" step inside an Orchestrator, or a stage in an `outputChannelName` chain. An **Internal Handler** connects any service in your Dependency Container to a named input channel so it can play the role of an [Endpoint](./), without exposing it as a Command Handler. If the service produces output, it can be connected to an output channel.&#x20;

### How to register

```php
class Shop
{
    #[InternalHandler("buyProduct")] 
    public function buyProduct(int $productId) : void
    {
        echo "Product with id {$productId} was bought";
    }
}
```

### Possible options

* `endpointId` - Endpoint identifier&#x20;
* `inputChannnelName` - Required option, defines to which channel endpoint should be connected
* `outputChannelName` - Channel where result of method invocation will be&#x20;
* `requiredInterceptorNames` - List of [interceptor](../../../modelling/extending-messaging-middlewares/interceptors/) names, which should intercept the endpoint
