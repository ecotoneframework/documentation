---
description: Repository PHP
---

# Repositories Introduction

Read [Aggregate Introduction](../state-stored-aggregate/) first for more details about Aggregates.

## The Problem

Every Command Handler does the same three lines: `findById`, `do something`, `save`. After ten handlers, that's thirty lines of identical glue. And the moment your aggregate becomes a Doctrine entity (or an Eloquent model), persistence concerns start leaking into the domain — fields exist to please the ORM, tests need a real database, and migrating to a different storage means rewriting every aggregate.

## How Ecotone Solves It

A **Repository** is the seam between your domain (which only knows aggregates) and your storage (Doctrine, Eloquent, Document Store, Event Store). Ecotone provides built-in repositories for each — pick one, point Ecotone at it, and your aggregates stay free of `@ORM\Entity` and `extends Model`. The fetch/save boilerplate disappears entirely; Ecotone wraps every command in load → call → save automatically.

## Typical Aggregate Flow

Repositories retrieve and save the aggregate to persistent storage. The typical flow for calling an aggregate method looks like:

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

By setting up `Repository` we provide Ecotone with functionality to fetch and store the Aggregate , so we don't need to write the above orchestration code anymore.

## Ecotone's Aggregate Flow

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

Now when we will send the Command, Ecotone will use ticketId from the Command to fetch related Ticket Aggregate, and will called **assignWorker passing the Command.** After this is completed it will use the repository to store changed Aggregate instance.

Therefore from high level nothing changes:

```php
$this->commandBus->send(
   new AssignWorkerCommand(
      $ticketId, $workerId,            
   )
);
```

This way we don't need to write orchestration level code ourselves.

