---
description: Splitter PHP
---

# Splitter

You receive a CSV with 5,000 orders. You could loop in your controller and call `commandBus->send()` 5,000 times — but the whole loop runs in one HTTP request, fails atomically, and one bad row blocks the rest. A **Splitter** takes the array as a Message and emits 5,000 individual Messages onto a downstream channel. Each row becomes its own retryable, individually-deduplicated unit of work.

Reach for a Splitter whenever the input is "one envelope, many work items" — bulk imports, fanning a webhook into per-line handlers, breaking up a daily report into per-customer jobs.&#x20;

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
