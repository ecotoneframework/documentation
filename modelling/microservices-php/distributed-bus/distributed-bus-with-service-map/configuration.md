# Configuration

## Service Name Configuration

In order for `Ecotone` how to route messages you need to register Service Name (Application Name).

* [Symfony Service Name Configuration](../../../../modules/symfony/symfony-ddd-cqrs-event-sourcing.md#servicename)
* [Laravel Service Name Configuration](../../../../modules/laravel/laravel-ddd-cqrs-event-sourcing.md#servicename)
* [Ecotone Lite Service Name Configuration](../../../../modules/ecotone-lite/#servicename)&#x20;

## Register Service Map

Register Distributed Bus with given Service Map:

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::createEmpty()
              ->withServiceMapping(
                        serviceName: "ticketService", 
                        channelName: "distributed_ticket_service"
              )
}
```

and define implementation of the distributed Message Channel:

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return SqsBackedMessageChannelBuilder::create("distributed_ticket_service")
}
```

For concrete use case, read [Main Section](./) or [Custom Features](custom-features.md) section.
