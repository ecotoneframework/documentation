---
description: Service Activator PHP
---

# Internal Message Handler

The Internal Handler connecting any service available in Depedency Container to an input channel so that it may play the role of a [Endpoint](./). If the service produces output, it may also be connected to an output channel.&#x20;

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
