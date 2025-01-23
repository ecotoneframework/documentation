---
description: Testing asynchronous communication in PHP
---

# Testing Asynchronous Messaging

When your code becomes asynchronous, sending Command is not enough to verify full flow. \
Your message will land in [pollable message channel](../../messaging/messaging-concepts/message-channel.md) (`queue`) awaiting for consumption. \
This requires executing your consumer in order to test full flow.&#x20;

`Ecotone` provides full support for testing your `asynchronous messaging architecture`.

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

## Running code Synchronously

By default all the asynchronous code will run synchronously. This simplifies the tested code and speed ups your test suite.

```php
$ecotoneTestSupport = EcotoneLite::bootstrapFlowTesting(
    // 1. We have dropped ChannelConfiguration::class, to replace it with our In Memory
    [OrderService::class, NotificationService::class],
    [new OrderService(), new NotificationService()]
);

// this will publish OrderWasPlaced as a result
$ecotoneTestSupport->sendCommandWithRoutingKey('order.register', new PlaceOrder('123'));

$this->assertEquals(
    1,
    count($this->notifier->getNotificationsOf('placedOrder'))
);
```

## Running Asynchronous Consumer

Ecotone provides `In Memory Pollable Channels` which can replace real implementation for testing purposes.

```php
$ecotoneTestSupport = EcotoneLite::bootstrapFlowTesting(
    [OrderService::class, NotificationService::class],
    [new OrderService(), new NotificationService()],
    // 1. we need to provide Message Channel to use
    enableAsynchronousProcessing: [
        SimpleMessageChannelBuilder::create('notifications')
    ]
);

// you could run Event Bus with OrderWasPlaced here instead
$ecotoneTestSupport->sendCommandWithRoutingKey('order.register', new PlaceOrder('123'));

// 2. running consumer
$ecotoneTestSupport->run('notifications');

$this->assertEquals(
    1,
    // 3. we can provide some in memory implementation for testing purposes
    count($this->notifier->getNotificationsOf('placedOrder'))
);
```

1. **Enable asynchronous processing -** We enable asynchronous processing and provide Message Channel to poll from. Message Channel can be Real (SQS, RabbitMQ, Dbal etc) or In Memory one
2. **Run** - This runs the the consumer with given PollingMetadata
3. **Assert** - We assert the state after consumer has exited

{% hint style="info" %}
In above example we are running consumer within same process as test. \
You may run consumer from separate process like this: (example for symfony):\
`php bin/console ecotone:run notifications --handledMessageLimit=1 --executionTimeLimit=100 --stopOnFailure`\
\
However running consumer as separate process is not advised, as it `requires booting separate process`which slows test suite, and due to lack of`shared memory` does not allow for using `In Memory implementations.`
{% endhint %}

## Default Message Channels

For testing with In Memory Channels, we can omit providing specific implementation. \
Ecotone will deliver an default In Memory Message Channels for us:

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
By default In Memory Queue Channel will do the serialization to PHP native serialization or your default Serialization if defined. This way it works in similar way to your production Queue Channels. \
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
