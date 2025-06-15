---
description: Repository PHP
---

# Repositories Introduction

Read [Aggregate Introduction](../state-stored-aggregate/) sections first to get more details about Aggregates.

## Typicial Aggregate Flow

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

