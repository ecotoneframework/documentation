# Concurrency Handling

Concurrency exceptions when multiple processes or threads access and modify shared resources simultaneously. These exceptions happen because two or more operations conflict try to change same piece of data. Ecotone provides built-in support for concurrency handling.&#x20;

In order to solve concurrent access, Ecotone implements Optimistic Locking.&#x20;

## Optimistic Locking

Each Event Sourcing _Aggregate_ or Event Sourcing _Saga_ has a version property that represents the current version of the resource. When modifications are made, the version is incremented. If two concurrent processes attempt to modify the same resource with different versions, a concurrency exception is raised. This is default behaviour, if we are using inbuilt Event Sourcing support.

### Custom Repositories

In case of Custom Repositories, we may use Ecotone support for optimistic locking to raise the exception in the Repository.

```php
public function save(
    array $identifiers, object $aggregate, array $metadata, 
    // Version to verify before storing
    ?int $versionBeforeHandling
): void
```

Version will be passed to the repository, based on **#\[AggregateVersion]** property inside the Aggregate/Saga.

We don't need to deal with increasing those on each action. Ecotone will increase it in our Saga/Aggregate automatically. \
\
We may also use inbuilt trait to avoid adding property manually.

```php
#[Saga]
class OrderProcess
{
    use WithAggregateVersioning;
     
    (...)
```

## Self Healing

To handle concurrency exceptions and ensure the system can self-heal, Ecotone offers retry mechanisms.&#x20;

_In synchronous scenarios_, like Command Handler being called via HTTP, [instant retries](retries.md#instant-retries) can be used to recover. If a concurrency exception occurs, the Command Message will be retried immediately, minimizing any impact on the end user. This immediate retry ensures that the Message Handler can self-heal and continue processing without affecting the user experience.\
\
&#xNAN;_&#x49;n asynchronous scenarios_, you can use still use instant retries, yet you may also provide [delayed retries](retries.md#delayed-retries). This means that when concurrency exception will occur, the Message will be retried after a certain delay. This as a result free the system resources from continues retries and allows for recovering after given period of delay.
