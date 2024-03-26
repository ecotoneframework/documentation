---
description: Testing Event Sourcing applications in PHP
---

# Testing Event Sourcing Applications

Ecotone comes with test support for testing Event Sourcing Applications. It provides sensible default and In Memory Event Store and Projection states, so you can implement each part of your event sourced application.

## Testing Aggregates

Testing aggregates and checking `expected events` was described in [previous chapter](testing-aggregates-and-sagas-with-message-flows.md), however if you would like to extend testing by providing initial `list of event`, you may do it:

```php
/** Setting up event sourced aggregate initial events */
$this->assertEquals(
    'Andrew',
    EcotoneLite::bootstrapFlowTesting([Ticket::class])
        // 1. setting up initial events for aggregate
        ->withEventsFor($ticketId, Ticket::class, [
            new TicketWasRegistered($ticketId, 'Johny', 'alert'),
            new AssignedPersonWasChanged($ticketId, 'Elvis'),
        ])
        ->sendCommand(new ChangeAssignedPerson($ticketId, 'Andrew'))
        ->getAggregate(Ticket::class, $ticketId)
        ->getAssignedPerson()
);
```

1. By using `withEventsFor` you may provide initial list of events for given aggregate

{% hint style="success" %}
In Ecotone Flow Testing you're not bounded to given aggregate instance. You may actually run `withEventsFor` for different identifiers or even different aggregates.\
This is especially useful, when testing `Projections`.
{% endhint %}

## Testing Aggregates with Event Store

You may also test your Event Sourced Aggregates and Sagas with In Memory Event Store that will `serialize` and `deserialize`, your events.&#x20;

{% hint style="success" %}
Testing with In Memory Event Store, ensures your events will be correctly serialized and deserialized. This make the tests more production like and covers your serialization and deserialization with tests.
{% endhint %}

Let's set up Ecotone Lite for this scenario:

```php
$this->assertEquals(
    'Andrew',
    // 1. Bootstrap with Event Store
    EcotoneLite::bootstrapFlowTestingWithEventStore(
        // 2. Set up Converters classes to resolve
        [Ticket::class, TicketEventConverter::class],
        // 2. Set up Converters instances
        [new TicketEventConverter()]
    )
        ->withEventsFor($ticketId, Ticket::class, [
            new TicketWasRegistered($ticketId, 'Johny', 'alert'),
            new AssignedPersonWasChanged($ticketId, 'Elvis'),
        ])
        ->sendCommand(new ChangeAssignedPerson($ticketId, 'Andrew'))
        ->getAggregate(Ticket::class, $ticketId)
        ->getAssignedPerson()
);
```

We provide list of converters, you may provide your [custom serialization](../../messaging/conversion/conversion.md) too.\
We need to set up repository that we will be using for tests.

1. When using `bootstrapFlowTestingWithEventStore` Ecotone will bootstrap In Memory Event Store for you and provide default configuration for your best development experience.
2. In case we have some customer converters, this is the place where we can set up them.

{% hint style="success" %}
If you're using default serialization mechanism ecotone/jms-converter, it will be loaded for you, so all you will need to register is custom Converters, if you have any.
{% endhint %}

## Testing Projections

Projections are following your streams and building read model. \
Ecotone allows you to rebuild your projections in synchronous way for testing purposes and can keep all the projection's state (like position) in memory.

{% hint style="info" %}
Testing projections works only with Event Store, so you need to make use of `bootstrapFlowTestingWithEventStore`.
{% endhint %}

```php
$this->assertEquals(
    [$productId->toString() => $productPrice],
    EcotoneLite::bootstrapFlowTestingWithEventStore(
            // 1. Setting projection and aggregate that we want to resolve
            [CurrentBasketProjection::class, Basket::class],
            [new CurrentBasketProjection(), new EmailConverter(), new PhoneNumberConverter(), new UuidConverter()],
        )
        // 2. Providing initial events to run projection on
        ->withEventsFor($userId, Basket::class, [
            new ProductWasAddedToBasket($userId, $productId, $productPrice)
        ])
        // 3. Triggering projection
        ->triggerProjection("current_basket")
        // 4. Runing query on projection to validate the state
        ->sendQueryWithRouting(CurrentBasketProjection::GET_CURRENT_BASKET_QUERY, $userId)
);
```

1. Setting up projection and related aggregate
2. Providing initial events that projection will use
3. After setting up initial events, we may trigger projection
4. And then we can run query, that will fetch projection's read model

{% hint style="success" %}
If your projection is asynchronous event driven, then for testing purposes by default it will become synchronous. In this way Ecotone helps in making your tests more readable.
{% endhint %}

## Acceptance Tests

Testing is flexible and not bounded to given class, you may test full application flow, if you want to.

```php
$this->assertEquals(
    [$productId->toString() => $productPrice],
    $this->getTestSupport($emailToken, $phoneNumberToken)
        ->sendCommand(new AddProduct($productId, "Milk", $productPrice))
        ->sendCommand(new RegisterUser($userId, "John Snow", Email::create('test@wp.pl'), PhoneNumber::create('148518518518')))
        ->sendCommand(new VerifyEmail($userId, VerificationToken::from($emailToken)))
        ->sendCommand(new VerifyPhoneNumber($userId, VerificationToken::from($phoneNumberToken)))
        ->sendCommand(new AddProductToBasket($userId, $productId))
        ->sendQueryWithRouting(CurrentBasketProjection::GET_CURRENT_BASKET_QUERY, $userId)
);
```

On this level we don't even need to interact with Projections, Events, Aggregates directly. \
We call set of commands and then using an query we can assert the state of the system.

{% hint style="success" %}
Acceptance tests are the high level tests that rarely change. They focus on interaction with the system like, it would be done by the end user.\
If acceptance tests pass, it ensures that end-users will be able to work with your system correctly.
{% endhint %}

## Testing with Real Event Store

So far we have been testing using In Memory Event Store, however we can run the test with real event store, if we want to be closer to production run.

```php
$ecotoneLite = EcotoneLite::bootstrapFlowTestingWithEventStore(
            [Ticket::class],
            // Provide connection to the database
            [DbalConnectionFactory::class => new DbalConnectionFactory('pgsql://ecotone:secret@localhost:5432/ecotone')],
            // Tell Ecotone to enable production Event Store
            runForProductionEventStore: true
);
```

However as it runs against real database instance we may require cleaning things up before each test.

```php
// delete event stream and reset projection
$ecotoneLite->deleteEventStream(Ticket::class);
$ecotoneLite->resetProjection("ticket_list_projection");
```
