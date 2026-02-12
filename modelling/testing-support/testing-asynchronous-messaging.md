---
description: Testing asynchronous communication in PHP
---

# Testing Asynchronous Messaging

When we make our code asynchronous, simply sending a command isn't enough to test the complete flow. The message lands in a message queue, waiting to be processed. To test the full workflow, we need to run the asynchronous consumer that pulls messages from the channel and executes our handlers.

Ecotone provides full testing support for asynchronous messaging, letting us verify end-to-end flows with real message brokers or using in memory implementation. This way we can test async behavior the same way we test synchronous code—keeping our tests fast, reliable, and easy to maintain.

## Example Asynchronous Handler

As an example, let's imagine scenario, where after placing order we want to send notification asynchronously.

```php
class NotificationService
{
    #[Asynchronous('notifications')]
    #[EventHandler(endpointId: 'notifyOrderWasPlaced')]
    public function notify(OrderWasPlaced $event, Notifier $notifier): void
    {
        $notifier->notifyAbout('placedOrder', $event->getOrderId());
    }
}
```

## Running Asynchronous Consumer

Ecotone provides support for testing asynchronous scenarios using Ecotone Lite:

```php
$ecotoneTestSupport = EcotoneLite::bootstrapFlowTesting(
    [OrderService::class, NotificationService::class],
    [new OrderService(), new NotificationService()],
    // 1. we need to provide Message Channel to use
    enableAsynchronousProcessing: [
        // In this scenario we are using In Memory implementation
        SimpleMessageChannelBuilder::create('notifications')
    ]
);

// you could run Event Bus with OrderWasPlaced here instead
$ecotoneTestSupport->sendCommandWithRoutingKey('order.register', new PlaceOrder('123'));

// 2. running consumer
$ecotoneTestSupport->run('notifications');

$this->assertEquals(
    1,
    // 3. asserting the result
    count($this->notifier->getNotificationsOf('placedOrder'))
);
```

1. **Enable asynchronous processing -** We enable asynchronous processing and provide Message Channel to poll from. Message Channel can be Real (SQS, RabbitMQ, Dbal etc) or In Memory one
2. **Run** - This runs the the consumer (test is still kept synchronous, therefore all debugging will work)
3. **Assert** - We assert the state after consumer has exited

{% hint style="success" %}
In the example above, we run the consumer within the same process as our test. We can also run the consumer from a separate process like this (example for Symfony):

`php bin/console ecotone:run notifications --handledMessageLimit=1 --executionTimeLimit=100 --stopOnFailure`

However, we don't recommend running the consumer as a separate process for testing. Here's why: it requires booting up a new PHP process for each test, which significantly slows down your test suite. More importantly, separate processes don't share memory, which means we can't use in-memory implementations, ensuring Message Consumers close correctly, and making debugging process much more difficult. By keeping everything in the same process, our tests run in milliseconds instead of seconds.
{% endhint %}

## Default Message Channels

For testing, we don't need to configure any specific message channel implementation. Ecotone automatically provides in-memory message channels for us. This means our tests run instantly without needing RabbitMQ, Redis, or any external services—perfect for fast, isolated unit tests that we can run anywhere, even on CI/CD pipelines without extra setup.

```php
$ecotoneTestSupport = EcotoneLite::bootstrapFlowTesting(
    [OrderService::class, NotificationService::class],
    [new OrderService(), new NotificationService()],
    // We simply state, that we want to enable async processing
    enableAsynchronousProcessing: true
);
```

## Polling Metadata

By default Ecotone will optimize for your test scenario:

* If **real Message Channel** like RabbitMQ, SQS, Redis will be used in test, then Message Consumer will be running up to **100ms** and will **stop on error**.
* If **In Memory Channel** will be used, then Message Consumer will be running **till it fetches all Messages** or **error will happen**.

The above default configuration ensures, tests will be kept stable and will run finish quickly.\
However if in case of need this behaviour can be customized by providing `ExecutionPollingMetadata`.

```php
$ecotoneTestSupport->run(
    'notifications',
    ExecutionPollingMetadata::createWithTestingSetup(
        // consumer will stop after handling single message
        amountOfMessagesToHandle: 1, 
        // or consumer will stop after 100 ms
        maxExecutionTimeInMilliseconds: 100,
        // or consumer will stop immediately after error
        failAtError: true
    )
);
```

## Testing Serialization

To test serialization we may fetch Message directly from the Channel and verify it's payload.

```php
$ecotoneTestSupport = EcotoneLite::bootstrapFlowTesting(
    [OrderService::class, NotificationService::class],
    [new OrderService(), new NotificationService()],
    enableAsynchronousProcessing: [
        // 1. Enable conversion on given channel
        SimpleMessageChannelBuilder::createQueueChannel(
            'notifications',
            conversionMediaType: 'application/json'
        )    
    ]
);

$ecotoneTestSupport->sendCommandWithRoutingKey('order.register', new PlaceOrder('123'));

$this->assertEquals(
    ['{"orderId":"123"}'],
    // 4. Verifing serialization - Get Event's payload from channel
    $ecotoneTestSupport->getMessageChannel('notifications')->receive()->getPayload()
);
```

1. We can enable serialization on this channel for given `Media Type`. In this case, we say serialize to `json` all message going through `notifications`.
2. We pull and verify messages sent to `notifications` channel, if their were sent in `json` format

{% hint style="success" %}
By default In Memory Queue Channel will do the serialization to PHP native serialization or your default Serialization if defined. This way it works in similar way to your production Queue Channels.\
If you don't want to use serialization however, you may set type to `conversionMediaType: MediaType::createApplicationXPHP()`
{% endhint %}

If our serialization mechanism is JMS Module, we will need to enable it for testing:

```php
$ecotoneTestSupport = EcotoneLite::bootstrapFlowTesting(
    [OrderService::class, NotificationService::class],
    [new OrderService(), new NotificationService()],
    configuration: ServiceConfiguration::createWithDefaults()
                ->withSkippedModulePackageNames(ModulePackageList::allPackagesExcept([
                        ModulePackageList::ASYNCHRONOUS_PACKAGE,
                        ModulePackageList::JMS_CONVERTER_PACKAGE
                ]))
    enableAsynchronousProcessing: [
        // 1. Enable conversion on given channel
        SimpleMessageChannelBuilder::createQueueChannel(
            'notifications',
            conversionMediaType: 'application/json'
        )    
    ]
);
```

Otherwise we will need to include Classes that customize our serialization.

## Testing Delayed Messages

Our Handlers may be delayed in time and we may want to run peform few actions and then release the message, to verify production like flow.

```php
class NotificationService
{
    #[Asynchronous('notifications')]
    #[Delayed(1000 * 60)] // 60 seconds
    #[EventHandler(endpointId: 'notifyOrderWasPlaced')]
    public function notify(OrderWasPlaced $event, Notifier $notifier): void
    {
        $notifier->notifyAbout('placedOrder', $event->getOrderId());
    }
}
```

```php
$ecotoneTestSupport = EcotoneLite::bootstrapFlowTesting(
    [OrderService::class, NotificationService::class],
    [new OrderService(), new NotificationService()],
    enableAsynchronousProcessing: [
       // 1. Turn on Delayable In Memory Pollable Channel
       SimpleMessageChannelBuilder::createQueueChannel('notifications', true)
    ]
);

$ecotoneTestSupport
    ->sendCommandWithRoutingKey('order.register', new PlaceOrder('123'));

// 2. Releasing messages awaiting for 60 seconds
$ecotoneTestSupport->run(
    'orders', 
    releaseAwaitingFor: TimeSpan::withSeconds(60)
);

$this->assertEquals(
    1,
    count($this->notifier->getNotificationsOf('placedOrder'))
);
```

1. The default behaviour for In Memory Channels is to ignore delays. By setting `second parameter` to `true` we are registering In Memory Channel that will be aware of delays.
2. We are releasing messages that awaits for 60 seconds or less.

### Delaying to given date

If we delay to given point in time, then we can use date time for releasing this message.

```php
$ecotoneTestSupport
    ->sendCommand(
        new SendNotification('123'),
        metadata: [
            "deliveryDelay" => new \DateTimeImmutable('+1 day')
        ]
    );

$ecotoneTestSupport->run(
    'notifications', 
    releaseAwaitingFor: new \DateTimeImmutable('+1 day')
);

$this->assertEquals(
    1,
    count($this->notifier->getNotifications())
);
```

### Changing Time

We can also change the time based on the needs

```php
// advance time to specific point in time
$ecotoneTestSupport
    ->changeTimeTo(new \DateTimeImmutable('2026-05-05 12:00:00')
    ->run('notifications');
    
$this->assertEquals(
    1,
    count($this->notifier->getNotifications())
);

// advance time by specific duration
$ecotoneTestSupport
    ->advanceTimeTo(Duration::minutes(60))
    ->run('notifications');
    
$this->assertEquals(
    1,
    count($this->notifier->getNotifications())
);
```

## Dropping all messages coming to given channel

In some scenarios, you may just want to turn off given channel, because you're not interested in messages that goes through it.

```php
$ecotoneTestSupport = EcotoneLite::bootstrapFlowTesting(
    // 1. We have dropped ChannelConfiguration::class, to replace it with our In Memory
    [OrderService::class, NotificationService::class],
    [new OrderService(), new NotificationService()],
    enableAsynchronousProcessing: [
        // 1. Create nullable channel
        SimpleMessageChannelBuilder::createNullableChannel('notifications')
    ]
);
```

1. By registering `nullable channel`, we make use that all messages that will go to given channel will be dropped.
