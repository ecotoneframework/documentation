---
description: Testing Ecotone messaging in Tempest applications
---

# Testing

Messaging behavior — including asynchronous, delayed and failing flows — is testable in-process with [Ecotone Lite](../../modelling/testing-support/), without booting Tempest, a database or a broker. Tests run in milliseconds.

## Test Mode and the Production Cache

The Tempest application skeleton sets `ENVIRONMENT=testing` in `phpunit.xml` — keep it. Ecotone reads the environment from `APP_ENV` or Tempest's `ENVIRONMENT` variable, and outside of `prod`/`production` it uses per-configuration caching, so your test runs never reuse the application's production cache.

To use [MessagingTestSupport](../../modelling/testing-support/testing-messaging.md) against a booted Tempest application, enable test mode in `ecotone.config.php`:

```php
return new EcotoneConfig(
    test: true,
);
```

## Flow Testing without Tempest

`EcotoneLite::bootstrapFlowTesting` runs your real handlers with in-memory infrastructure. Tempest services your handlers inject (like `Mailer`) are provided as stubs:

```php
$mailer = new class implements Mailer {
    public array $subjects = [];

    public function send(Email $email): void
    {
        $this->subjects[] = $email instanceof GenericEmail ? (string) $email->subject : '';
    }
};

$ecotone = EcotoneLite::bootstrapFlowTesting(
    classesToResolve: [OrderConfirmationWorkflow::class],
    containerOrAvailableServices: [
        new OrderConfirmationWorkflow(),
        Mailer::class => $mailer,
    ],
    enableAsynchronousProcessing: [
        SimpleMessageChannelBuilder::createQueueChannel('notifications', delayable: true),
    ],
);
```

{% hint style="warning" %}
When testing handlers that use `#[Delayed]`, create the in-memory channel with `delayable: true`. A plain queue channel ignores delays and delivers immediately, which silently turns a delayed-delivery test into a pass-through.
{% endhint %}

## Testing Delayed Messages with Time Travel

There is no need to sleep through real delays — release the channel's awaiting messages as if time had passed:

```php
$ecotone->publishEvent(new ShipmentWasDispatched(
    orderId: '7', customerName: 'Anna', customerEmail: 'anna@example.com',
));

// Run the channel now: the message is not due yet
$ecotone->run('notifications', ExecutionPollingMetadata::createWithTestingSetup());
$this->assertSame([], $mailer->subjects);

// Travel 31 seconds forward and run again
$ecotone->releaseAwaitingMessagesAndRunConsumer('notifications', 31_000);

$this->assertSame(['How was order #7?'], $mailer->subjects);
```

## Testing the Failure Path

Retries, dead-letter parking and failure isolation are ordinary flow tests. Configure the error handling the same shape as production, with an in-memory parking channel:

```php
$ecotone = EcotoneLite::bootstrapFlowTesting(
    classesToResolve: [OrderConfirmationWorkflow::class],
    containerOrAvailableServices: [new OrderConfirmationWorkflow(), Mailer::class => $failingMailer],
    configuration: ServiceConfiguration::createWithDefaults()
        ->withDefaultErrorChannel('errorChannel')
        ->withExtensionObjects([
            ErrorHandlerConfiguration::createWithDeadLetterChannel(
                'errorChannel',
                RetryTemplateBuilder::fixedBackOff(1)->maxRetryAttempts(2),
                'parked',
            ),
            SimpleMessageChannelBuilder::createQueueChannel('parked'),
        ]),
    enableAsynchronousProcessing: [
        SimpleMessageChannelBuilder::createQueueChannel('notifications', delayable: true),
    ],
);

$ecotone->publishEvent($orderForFailingRecipient);
$ecotone->publishEvent($orderForHealthyRecipient);

$ecotone->run('notifications', ExecutionPollingMetadata::createWithTestingSetup(failAtError: false));
$ecotone->releaseAwaitingMessagesAndRunConsumer('notifications', 10_000, ExecutionPollingMetadata::createWithTestingSetup(failAtError: false));
$ecotone->releaseAwaitingMessagesAndRunConsumer('notifications', 10_000, ExecutionPollingMetadata::createWithTestingSetup(failAtError: false));
$ecotone->releaseAwaitingMessagesAndRunConsumer('notifications', 10_000, ExecutionPollingMetadata::createWithTestingSetup(failAtError: false));

// The healthy message was delivered despite the poison one on the same channel
$this->assertSame(['anna@example.com'], $failingMailer->delivered);
// Initial attempt + two retries
$this->assertSame(3, count($failingMailer->attemptsFor('down@unreachable.example')));
// Exhausted retries park the message instead of losing it
$this->assertNotNull($ecotone->receiveMessageFrom('parked'));
```

Each `releaseAwaitingMessagesAndRunConsumer` call releases the next scheduled retry. "The queue retried, parked the message, and did not block the others" becomes an assertion instead of something you can only observe on staging infrastructure.

## Integration Tests that Boot Tempest

When a test boots a real Tempest kernel, clear Ecotone's static state **before** the kernel boots — discovery may compile the messaging system during boot, and clearing afterwards leaves a container without its registered gateways:

```php
protected function setUp(): void
{
    EcotoneServiceInitializer::clearCache();
    MessagingSystemInitializer::clearDefinitionHolder();

    // ...boot the Tempest kernel
}
```
