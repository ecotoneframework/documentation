# Testing

Distributed Bus with Service Map comes with ability to test using real or in memory Message Channels. We can test run the tests using [Ecotone Lite test support](testing.md).

## Testing Flow between Services

Suppose we do have two different Services **User Service** and **Ticket Service**. From User Service we would like to send Command to create new ticket, whenever something interesting happens.&#x20;

Therefore using Ecotone Lite we can roll out two different applications and share the channel to test the full flow. Let's start by defining Publishing Service (Application):

```php
# test file

$distributedTicketChannel = SimpleMessageChannelBuilder::createQueueChannel('distributed_ticket');

$userService = EcotoneLite::bootstrapFlowTesting(
            configuration: ServiceConfiguration::createWithDefaults()
                ->withServiceName('user_service')
                ->withExtensionObjects([
                    // distributed service map on the publisher side
                    DistributedServiceMap::initialize()
                        ->withServiceMapping(serviceName: 'ticket_service', channelName: 'distributed_ticket'),
                    // shared in memory channel
                    enableAsynchronousProcessing: [distributedTicketChannel]
                ]),
        )
```

When we do have Publishing Service, we can now set up Consumer side. Suppose we would like to test  such Distributed Command Handler:

```php
# application file

class TicketService
{
    #[Distributed]
    #[CommandHandler('ticket.create')]
    public function createTicket(CreateTicket $command, TicketRepository $ticketRepository): void
    {
        $ticket = Ticket::create($command);
        $ticketRepository->save($ticket);
    }
}
```

So let's define the Distributed Consumer

```php
# test file
(...)

$ticketRepository = new InMemoryTicketRepository();
$ticketService = EcotoneLite::bootstrapFlowTesting(
            classesToResolve: [TicketService::class],
            containerOrAvailableServices: [
                TicketService::class => new TicketService(),
                TicketRepository::class => $ticketRepository,            
            ]
            configuration: ServiceConfiguration::createWithDefaults()
                ->withServiceName('ticket_service'),
            enableAsynchronousProcessing: [distributedTicketChannel]
        )
```

As we do have Publishing and Consuming side ready, we can now send Command from User Service.

```php
$userService->getDistributedBus()->convertAndSendCommand(
    'ticket_service',
    'ticket.create',
    new CreateTicket(//data),
);
```

This as a result will send this Command Message to Distributed Message Channel from which we can consume and verify, if our Ticket was stored correctly:

```php
$ticketService->run('ticket_service');
$this->assertSame(
    1,
    $ticketRepository->count(),
);
```

## Testing with real Message Channels and serialization

We may also test out using real production Message Channels, it's simple as switching to different provider:

```php
$distributedTicketChannel = SqsBackedMessageChannelBuilder::create("distributed_ticket")
```

We can also test serialization using [In Memory Message Channels](../../../testing-support/testing-asynchronous-messaging.md#testing-serialization).
