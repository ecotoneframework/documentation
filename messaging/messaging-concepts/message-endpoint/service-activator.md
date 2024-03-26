---
description: Service Activator PHP
---

# Service Activator

The Service Activator connecting any service available in Depedency Container to an input channel so that it may play the role of a [Endpoint](./). If the service produces output, it may also be connected to an output channel. \
Alternatively, an output producing service may be located at the end of a processing pipeline or message flow in which case, the inbound Message's "replyChannel" header can be used. This is the default behavior if no output channel is defined.

### How to register

```php
class Shop
{
    #[ServiceActivator("buyProduct")] 
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
* `requiredInterceptorNames` - List of [interceptor](../../../modelling/extending-messaging-middlewares/interceptors.md) names, which should intercept the endpoint
