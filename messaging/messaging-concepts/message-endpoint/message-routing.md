---
description: Message Router PHP
---

# Message Router

Routers consume messages from a message channel and forward each consumed message to one or more different message channels depending on a defined conditions.

Router must return name of the channel, where the message should be routed too. It can be array of channel names, if there are more.&#x20;

```php
class OrderRouter
{
    #[Router("make.order")] 
    public function orderSpecificType(string $orderType) : string
    {
        return $orderType === 'coffee' ? "orderInCoffeeShop" : "orderInGeneralShop";
    }
}
```

### Possible options

* `endpointId` - Endpoint identifier&#x20;
* `inputChannnelName` - Required option, defines to which channel endpoint should be connected
* `isResolutionRequired` - If true, will throw exception if there was no channel name returned

## Routing to multiple Message Channels

```php
class OrderRouter
{
    #[Router("order.bought")] 
    public function distribute(string $order) : array
    {
        // list of Channel names to distribute Message too
        return [
            'audit.store',
            'notification.send',
            'order.close'
        ];
    }
}
```

## What can be Router used for?  &#x20;

Router is powerful concept that is backing up Query/Command and Event Bus implementations. \
Together with [Message Gateway](../messaging-gateway.md#implementing-own-gateway), you may roll up your own Bus implementation or build workflow pipelines.&#x20;

### Own Bus implementation

```php
interface QueryGateway
{
    #[MessageGateway("query.router")]
    public function query(string $queryEndpoint): string;
}

final class QueryService
{
    #[Router("query.router")]
    public function getQuery(string $shopName): string
    {
        return sprintf("query-%s-shop", $shopName);
    }
}

final class ShopQueryHandler
{
    #[QueryHandler("query-coffee-shop")]
    public function queryOne(): string
    {
        return "coffee";
    }

//  We are routing directly to given channel name, so we use lower level abstraction ServiceActivator
//  The benefit of it is, that endpoint is actually hidden and can not be called directly from QueryBus.
    #[ServiceActivator("query-milk-shop")]
    public function queryTwo(): string
    {
        return "milk";
    }
}
```
