---
description: Testing Aggregate, Saga in PHP
---

# Testing Aggregates and Sagas with Message Flows

On top of [Messaging Test Support](testing-messaging.md), Ecotone provides abstraction to handle Message Flows.\
This is a clean way to define step by step what happens in your system and verify the results.&#x20;

{% hint style="success" %}
Testing with Message Flows, make your code closer to production like execution.\
Message Flows are great candidate for writing acceptance tests or testing flow of your Aggregates and Sagas.
{% endhint %}

## Setting up Flow Test

To enable Flow Test Support, as first parameter we pass list of classes we want to resolve for our testing scenario, this can be `aggregate`, `saga` or any class containing Ecotone's `attributes`.

```php
$testSupport = EcotoneLite::bootstrapFlowTesting([User::class]);
```

{% hint style="success" %}
For more details on how to set up EcotoneLite for the different test scenario, read introduction chapter [Testing Messaging](testing-messaging.md).
{% endhint %}

## Testing Aggregate or Sagas

It does not matter, if we test Saga or Aggregate, as the case will looks similar.\
So as an example, let's take Aggregate that looks like this:

```php
#[Aggregate]
final class User
{
    use WithEvents;

    public function __construct(
        #[Identifier] private UuidInterface $userId,
        private string $name,
        private Email $email,
        private PhoneNumber $phoneNumber,
        private $isBlocked = false,
        private $isVerified = false
    ) {
        $this->recordThat(new UserWasRegistered($this->userId, $this->email, $this->phoneNumber));
    }

    #[CommandHandler]
    public static function register(RegisterUser $command): self
    {
        return new self(
            $command->getUserId(),
            $command->getName(),
            $command->getEmail(),
            $command->getPhoneNumber()
        );
    }

    #[CommandHandler("user.block")]
    public function block(): void
    {
        $this->isBlocked = true;
    }
    
    #[CommandHandler("user.verify")]
    public function verify(VerifyUser $command): void
    {
        $this->isVerified = true;
    }

    public function isBlocked(): bool
    {
        return $this->isBlocked;
    }
}
```

When we've our Ecotone Lite set up, we may start using flow testing:

```php
public function test_registering_user()
{
    $userId = Uuid::uuid4();
    $email = Email::create("test@wp.pl");
    $phoneNumber = PhoneNumber::create("148518518518");

    // 1. Comparing published events after registration
    $this->assertEquals(
        [new UserWasRegistered($userId, $email, $phoneNumber)],
        // 2. Running Message Flow Test support
        $testSupport
            ->sendCommand(new RegisterUser($userId, "johny", $email, $phoneNumber))
            ->getRecordedEvents()
    );
}   
```

1. We set up our expected event as outcome of running `RegisterUser Command.`
2. Then we can make use of Flow Support to send an Command and get event that was recorded as outcome.

{% hint style="success" %}
You may send multiple command and chain them together, to build flow that you would like to test.
{% endhint %}

## Verifying Aggregate's State

In cases when you're not using Event Sourced Aggregate, you may want to test state of the aggregate. We work only with high level API which are commands and we don't have direct access to Aggregate, `Ecotone` however exposes a possibility to fetch aggregate inside the flow.

```php
$this->assertTrue(
    $testSupport
        ->sendCommand(new RegisterUser($userId, "johny", $email, $phoneNumber))
        // 1. Command with routing key
        ->sendCommandWithRoutingKey("user.block", metadata: ["aggregate.id" => $userId])
        // 2. Fetching aggregate
        ->getAggregate(User::class, $userId)
        // 3. Calling aggregate method
        ->isBlocked()
);
```

1. This `Command Handler` was registered by routing key without Command. With Flow support we may call Buses using routing key, and to tell which aggregate we want to call, we may use `aggregate.id` metadata.
2. Then we fetch given aggregate by id
3. And after that we can call method on aggregate that we've fetched

If you mark your query method with `QueryHandler` attribute it becomes part of the flow, and we don't need to add `->addAggregateUnderTest(User::class)` and we may write following test instead:

```php
#[QueryHandler("user.isBlocked")]
public function isBlocked(): bool
{
    return $this->isBlocked;
}
```

```php
$this->assertTrue(
    $testSupport
        ->sendCommand(new RegisterUser($userId, "johny", $email, $phoneNumber))
        ->sendCommandWithRoutingKey("user.block", metadata: ["aggregate.id" => $userId])
        ->sendQueryWithRouting('user.isBlocked', metadata: ["aggregate.id" => $userId])
);
```

## Testing Communication between Aggregates and Sagas.

With `Ecotone Lite` you may create full acceptance tests, just add classes that should be used in your test scenarios and let Ecotone glue everything together.

```php
public function test_success_verification_after_registration()
{
    $testSupport = EcotoneLite::bootstrapFlowTesting(
    // 1. We are resolving User aggregate and Verification Process Saga
        [User::class, VerificationProcess::class],
        enableAsynchronousProcessing: [
            SimpleMessageChannelBuilder::createQueueChannel("asynchronous_messages", true)        
        ]
    );

    $userId = Uuid::uuid4();

    $this->assertEquals(
        [['user.block', $userId->toString()]],
        $testSupport
            ->sendCommand(new RegisterUser($userId, "John Snow", Email::create('test@wp.pl'), PhoneNumber::create('148518518518')))
            ->discardRecordedMessages()
            ->run("asynchronous_messages", releaseAwaitingFor: TimeSpan::withHours(24))
            ->getRecordedCommandsWithRouting()
    );
}
```

{% hint style="success" %}
For given scenario you may add set of classes that should be used in this test.\
You may include `Interceptors`, `Converters`, too. \
\
In general all your Ecotone's production code will work in your test scenarios.\
This ensures tests will be as close to production as it's possible.
{% endhint %}

## Testing Output Channels

When we want to test given Aggregate or Sagas, yet we want to do it in isolation and we do use of Ecotone's [Input/Output Pipes support](../business-workflows/). Then we may want to verify if given Message was send to given channel. Consider below example:

```php
#[Saga]
final class OrderProcessSaga
{
    use WithEvents;

    public function __construct(
        #[Identifier] private string $orderId,
    )
    {
        $this->recordThat(new OrderProcessSagaStarted($orderId));
    }

    #[EventHandler]
    public static function whenOrderWasPlaced(OrderWasPlaced $event): self
    {
        return new self($event->orderId);
    }

    #[EventHandler(outputChannelName: "takePayment")]
    public function triggerPayment(OrderProcessSagaStarted $event): TakePayment
    {
        return new TakePayment($event->orderId);
    }
}
```

When Saga was started we will send an **instance of TakePayment** to "**takePayment"** output channel.\
However if our test suite does not include given Message Handler and we still want to hook to get if given Message was sent, we can do it as below:

```php
public function test_workflow_with_separated_output_command_handler(): void
{
    $ecotoneLite = EcotoneLite::bootstrapFlowTesting(
        [OrderProcessSaga::class]
    );

    $orderId = '123';

    $this->assertEquals(
        [new TakePayment($orderId)],
        $ecotoneLite
            ->publishEvent(new OrderWasPlaced($orderId))
            // This will fetch Message Payloads that have been sent to given channel name
            ->getRecordedMessagePayloadsFrom('takePayment')
    );
}
```

{% hint style="success" %}
Even if given Message is an Command from the perspective of outputChannel is just a simple Message. That's it won't be recorded as Command and we need to use `getRecordedMessagePayloadsFrom` instead.
{% endhint %}

## Testing with Real State-Stored Repository

By default Ecotone Lite provides for testing purposes In Memory Repositories.\
If you want to run your test against real database, you may disable this behaviour and provide your own repository.

```php
$testSupport = EcotoneLite::bootstrapFlowTesting(
// 1. We are resolving User aggregate
    [User::class, UserRepository::class],
    [UserRepository::class => $this->container->get(UserRepository::class)],
    addInMemoryStateStoredRepository: false
);
```

You may also disable Event Sourced Repository:

```php
addEventSourcedRepository: false
```
