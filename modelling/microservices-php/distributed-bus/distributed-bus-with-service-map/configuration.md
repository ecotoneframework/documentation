# Configuration

## Service Name Configuration

In order for `Ecotone` how to route messages you need to register Service Name (Application Name).

* [Symfony Service Name Configuration](../../../../modules/symfony/symfony-ddd-cqrs-event-sourcing.md#servicename)
* [Laravel Service Name Configuration](../../../../modules/laravel/laravel-ddd-cqrs-event-sourcing.md#servicename)
* [Ecotone Lite Service Name Configuration](../../../../modules/ecotone-lite/#servicename)&#x20;

## Register Service Map for Consumption

The minimum needed for enabling Distributed Bus with Service Map and start consuming is to tell Ecotone, that we do use Service Map within the Service

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize();
}
```

and then we would define Message Channel, which we would use for for incoming messages:

```php
#[ServiceContext]
public function channels()
{
    return SqsBackedMessageChannelBuilder::create("distributed_ticket_service")
}
```

## Register Service Map for Publishing

Register Distributed Bus with given Service Map:

```php
#[ServiceContext]
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withServiceMapping(
                        serviceName: "ticketService", 
                        channelName: "distributed_ticket_service"
              )
}
```

and define implementation of the distributed Message Channel:

```php
#[ServiceContext]
public function channels()
{
    return SqsBackedMessageChannelBuilder::create("distributed_ticket_service")
}
```

For concrete use case, read [Main Section](./) or [Custom Features](custom-features.md) section.
