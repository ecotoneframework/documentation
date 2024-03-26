---
description: Transactions, Asynchronous, Dead Letter Queue PHP DBAL
---

# DBAL Support

## Installation

```php
composer require ecotone/dbal
```

### Module Powered By

Powered by powerful database abstraction layer [Doctrine/Dbal](https://github.com/doctrine/dbal) and [Enqueue](https://php-enqueue.github.io/) for asynchronous communication&#x20;

## Configuration

To configure Connection follow instruction for given integration

* [Symfony Dbal Module](symfony/symfony-database-connection-dbal-module.md)
* [Laravel Dbal Module](laravel/database-connection-dbal-module.md)
* [Ecotone Lite](ecotone-lite/database-connection-dbal-module.md)

## Message Channel

To create Dbal Backed [Message Channel](../modelling/asynchronous-handling/), we need to create [Service Context](../messaging/service-application-configuration.md).&#x20;

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function orderChannel()
    {
        return DbalBackedMessageChannelBuilder::create("orders");
    }
}
```

Now `orders` channel will be available in our Messaging System.&#x20;

### Message Channel Configuration

```php
DbalBackedMessageChannelBuilder::create("orders")
    ->withAutoDeclare(false) // do not auto declare queue
    ->withDefaultTimeToLive(1000) // limit TTL of messages
```

## Transactions

By default `Ecotone`enables transactions for all [Asynchronous Endpoints](../tutorial-php-ddd-cqrs-event-sourcing/php-asynchronous-processing.md) and Command Bus. You may use of [`Service Context`](../messaging/service-application-configuration.md) to turn off this configuration. You may also add more connections to be handled.

```php
class DbalConfiguration
{
    #[ServiceContext]
    public function registerTransactions() : array
    {
        return [
            DbalConfiguration::createWithDefaults()
                ->withTransactionOnCommandBus(true) // Turn for running command bus
                ->withTransactionOnAsynchronousEndpoints(true) // for all asynchronous endpoints
                ->withoutTransactionOnAsynchronousEndpoints(["notifications"]) // turn off for list of asynchronous endpoint 
                ->withDefaultConnectionReferenceNames([
                    "Enqueue\Dbal\DbalConnectionFactory",
                    "AnotherDbalConnectionFactory"
                ])
        ];
    }

}
```

If we disable global transactions, it make sense to enable transactions on specific endpoint. \
To do it all we need to do is to mark it with `Ecotone\Dbal\DbalTransaction\DbalTransaction` attribute.

```php
#[CommandHandler]
#[DbalTransaction] 
public function sellProduct(SellProduct $command) : void
{
    // do something with $command
}
```

## Document Store

DBAL provides support for [Document Store](dbal-support.md#undefined), which is enabled by default.\
Every document is stored inside the "`ecotone_document_store`" table.&#x20;

### Standard Aggregate Repository

You may enable support for [storing standard aggregates](dbal-support.md#standard-aggregate-repository).

```php
    #[ServiceContext]
    public function getDbalConfiguration(): DbalConfiguration
    {
        return DbalConfiguration::createWithDefaults()
                ->withDocumentStore(enableDocumentStoreAggregateRepository: true);
    }
```

### In Memory Document Store&#x20;

For testing purposes you may want to enable `In Memory implementation`.

```php
    #[ServiceContext]
    public function configuration()
    {
        return DbalConfiguration::createWithDefaults()
                    ->withDocumentStore(inMemoryDocumentStore: true);
    }
```

### Table initialization

Table will be create for you, however this comes with extra SQL cost, to verify before adding new document, if table exists. \
After releasing you may want to disable the check, as you know, that the table already exists.

```php
    #[ServiceContext]
    public function getDbalConfiguration(): DbalConfiguration
    {
        return DbalConfiguration::createWithDefaults()
                ->withDocumentStore(initializeDatabaseTable: false);
    }
```
