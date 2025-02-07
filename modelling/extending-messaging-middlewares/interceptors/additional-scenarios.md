# Additional Scenarios

## Access attribute from interceptor

We may access attribute from the intercepted endpoint in order to perform specific action

```php
#[\Attribute]
class Cache 
{
    public string $cacheKey;
    public int $timeToLive;
    
    public function __construct(string $cacheKey, int $timeToLive)
    {
        $this->cacheKey = $cacheKey;
        $this->timeToLive = $timeToLive;
    }
}
```

then we would have an Message Endpoint using this Attribute:

```php
class ProductsService
{
   #[QueryHandler]
   #[Cache("hotestProducts", 120)]
   public function getHotestProducts(GetOrderDetailsQuery $query) : array
   {
      return ["orderId" => $query->getOrderId()]
   }
}  
```

and it can be used in the intereceptors by type hinting given parameter:

```php
class NotificationFilter
{
    #[After] 
    public function filter($result, Cache $cache) : ?array
    {
        $this->cachingSystem($cache->cacheKey, $result, $cache->timeToLive);
    }
}
```
