# Configure Repository

To use Ecotone's Aggregate functionality, we need a registered repository. Ecotone comes with built-in support for popular persistence options like Doctrine ORM, Eloquent, and document stores, so there's a good chance we can use what we already have without extra work. If our storage solution isn't supported yet, or if we have specific requirements, we can easily register our own custom repository by following the steps in this section. This flexibility means we're not locked into any particular database or ORM—we can use whatever fits our project best.

## Repository for State-Stored Aggregate

State-Stored Aggregate are normal `Aggregates`, which are stored using Standard Repositories. \
Therefore to configure Repository for your Aggregate, create a class that extends **StandardRepository** interface:

```php
interface StandardRepository
{
    
    1 public function canHandle(string $aggregateClassName): bool; 
    
    2 public function findBy(string $aggregateClassName, array $identifiers) : ?object;
    
    3 public function save(array $identifiers, object $aggregate, array $metadata, ?int $expectedVersion): void;
}
```

1. `canHandle method` informs, which `Aggregate Classes` can be handled with this `Repository`. Return true, if saving specific aggregate is possible, false otherwise.
2. `findBy method` returns if found, existing `Aggregate instance`, otherwise null.
3. `save method` is responsible for storing given `Aggregate instance`.

* `$identifiers` are array of `#[Identifier]` defined within aggregate.
* `$aggregate` is instance of aggregate
* `$metadata` is array of extra information, that can be passed with Command
* `$expectedVersion` if version locking by `#[Version]` is enabled it will carry currently expected

### Set up your own Implementation

When your implementation is ready simply mark it with `#[Repository]` attribute:

```php
#[Repository]
class DoctrineRepository implements StandardRepository
{
    // implemented methods
}
```

### Example implementation using Doctrine ORM

This is example implementation of Standard Repository using Doctrine ORM.

_**Repository:**_

```php
final class EcotoneTicketRepository implements StandardRepository
{
    public function __construct(private readonly EntityManagerInterface $entityManager)
    {
    }

    public function canHandle(string $aggregateClassName): bool
    {
        return $aggregateClassName === Ticket::class;
    }

    public function findBy(string $aggregateClassName, array $identifiers): ?object
    {
        return $this->entityManager->getRepository(Ticket::class)
                    // Array of identifiers for given Aggregate
                    ->find($identifiers['ticketId']);
    }

    public function save(array $identifiers, object $aggregate, array $metadata, ?int $versionBeforeHandling): void
    {
        $this->entityManager->persist($aggregate);
    }
}
```

## Using Multiple Repositories

By default Ecotone when we have only one Standard and Event Sourcing Repository registered, Ecotone will use them for our Aggregate by default.\
This comes from simplification, as if there is only one Repository of given type, then there is nothing else to be choose from.\
\
However, if we register multiple Repositories, then we need to take over the process and tell which Repository will be used for which Aggregate.

* In case of [Custom Repositories](configure-repository.md#set-up-your-own-implementation) we do it using **canHandle** method.
* In case of inbuilt Repositories, we should follow configuration section for given type

## Repository for Event Sourced Aggregate

Custom repository for Event Sourced Aggregates is described in more details under [Event Sourcing Repository section](../../event-sourcing/event-sourcing-introduction/persistence-strategy/event-sourcing-repository.md).
