# Access Event Store

## Use Event Store directly

In some cases you may want to access `Event Store` directly. \
Event Store is auto registered in your Dependency Container, so you can fetch it like any other service or inject it directly to any Handler.

```php
use Ecotone\EventSourcing\EventStore;
    
    #[QueryHandler(self::GET_CURRENT_BALANCE_QUERY)]
    public function getCurrentBalance(#[Reference] EventStore $eventStore): array
    {
        $streamName = "wallet";
        if (!$eventStore->hasStream($streamName)) {
            return [];
        }

        /** @var Event[] $event */
        $events = $eventStore->load($streamName, count: 10);

        return $events;
    }
```
