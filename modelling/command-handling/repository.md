---
description: Repository PHP
---

# Repositories Introduction

Read [Aggregate Introduction](state-stored-aggregate/) sections first to get more details about Aggregates.

## Fetching Aggregate from Repository

Repositories are used for retrieving and saving the aggregate to persistent storage. \
Typical flow for calling aggregate method would looks like below:

```php
class AssignWorkerHandler
{
    private TicketRepository $ticketRepository;

    #[CommandHandler]
    public function handle(AssignWorkerCommand $command) : void
    {
       // fetch the aggregate from repository
       $ticket = $this->ticketRepository->findBy($command->getTicketId());
       // call action method
       $ticket->assignWorker($command);
       // store the aggregate in repository
       $this->ticketRepository->save($ticket);    
    }
}
```

```php
$this->commandBus->send(
   new AssignWorkerCommand(
      $ticketId, $workerId,            
   )
);
```

By setting up `Repository` we provide Ecotone with functionality to fetch and store the Aggregate , so we don't need to do it on our own.

## How Repository is used

If our class is defined as Aggregate, Ecotone will use Repository in order fetch and store it, whenever the `Command` is sent via `Command Bus`.&#x20;

```php
#[Aggregate]
class Ticket
{
    #[Identifier]
    private string $ticketId;

    #[CommandHandler]
    public function assignWorker(AssignWorkerCommand $command)
    {
       // do something with assignation
    }
}
```

There are two types of repositories. One for storing [`State-Stored Aggregate`](state-stored-aggregate/#state-stored-aggregate) and another one for storing [`Event Sourcing Aggregate`](state-stored-aggregate/#event-sourcing-aggregate).

Based on which interface is implemented, `Ecotone` knows which Aggregate type was selected.

## Repository for State-Stored Aggregate

State-Stored Aggregate are normal `Aggregates`, which are not `Event Sourced`.

```php
interface StandardRepository
{
    
    1 public function canHandle(string $aggregateClassName): bool; 
    
    2 public function findBy(string $aggregateClassName, array $identifiers) : ?object;
    
    3 public function save(array $identifiers, object $aggregate, array $metadata, ?int $expectedVersion): void;
}
```

1. `canHandle method` informs, which `Aggregate Classes` can be handled with this `Repository`. Return true, if saving specific aggregate is possible, false otherwise.
2. `findBy method` returns if found, existing `Aggregate instance`, otherwise null.&#x20;
3. `save method` is responsible for storing given `Aggregate instance`.&#x20;

* `$identifiers` are array of `#[Identifier]` defined within aggregate.
* `$aggregate` is instance of aggregate
* `$metadata` is array of extra information, that can be passed with Command
* `$expectedVersion` if version locking by `#[Version]` is enabled it will carry currently expected&#x20;

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

## Repository for Event Sourced Aggregate

```php
interface EventSourcedRepository
{
    public function canHandle(string $aggregateClassName): bool;
    
    1 public function findBy(string $aggregateClassName, array $identifiers) :  EventStream;

    2 public function save(array $identifiers, string $aggregateClassName, array $events, array $metadata, int $versionBeforeHandling): void;
}
```

Event Sourced Repository  instead of working with aggregate instance, works with events.&#x20;

1. `findBy method` returns previously created events for given aggregate.&#x20;
2. `save method` gets array of events to save returned by `CommandHandler` after performing an action

### Set up your own Implementation

When your implementation is ready simply mark it with `#[Repository]` attribute:

```php
#[Repository]
class DoctrineRepository implements EventSourcedRepository
{
    // implemented methods
}
```

## Inbuilt Repositories

Ecotone provides inbuilt repositories to get you started quicker. This way you can enable given repository and start implementing higher level code without worrying about infrastructure part.

### - Doctrine ORM Support

This provides integration with [Doctrine ORM](https://www.doctrine-project.org/projects/orm.html). To enable it read more in [Symfony Module Section](../../modules/symfony/doctrine-orm.md).

### - Laravel Eloquent Support

This provides integration with [Eloquent ORM](https://laravel.com/docs/5.0/eloquent/). Eloquent support is available out of the box after installing [Laravel module](../../modules/laravel/laravel-ddd-cqrs-event-sourcing.md).

### - Document Store Repository

This provides integration [Document Store](../../messaging/document-store.md) using relational databases. It will serialize your aggregate to json and deserialize on load using [Converters](../../messaging/conversion/conversion/).\
To enable it read in [Dbal Module Section](../../modules/dbal-support.md#document-store).

### - Event Sourcing Repository

Ecotone provides inbuilt Event Sourcing Repository, which will set up Event Store and Event Streams. To enable it read [Event Sourcing Section](../event-sourcing/).
