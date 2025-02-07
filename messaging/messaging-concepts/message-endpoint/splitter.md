---
description: Splitter PHP
---

# Splitter

The Splitter is endpoint where message can be splitted in several parts and be sent to be processed indepedently.&#x20;

```php
class Shop
{
    #[Splitter(inputChannelName="buyProduct", outputChannelName="buySingleProduct")]
    public function sendMultipleOrders(array $products) : array
    {
        return $products;
    }

    #[ServiceActivator("buySingleProduct")] 
    public function buyProduct(string $productName) : void
    {
        echo "Product {$productName} was bought";
    }
}
```

### Possible options

* `endpointId` - Endpoint identifier&#x20;
* `inputChannnelName` - Required option, defines to which channel endpoint should be connected
* `outputChannelName` - Channel where result of method invocation will be&#x20;
* `requiredInterceptorNames` - List of [interceptor](../../../modelling/extending-messaging-middlewares/interceptors/) names, which should intercept the endpoint
