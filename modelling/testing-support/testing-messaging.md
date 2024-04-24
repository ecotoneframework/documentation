---
description: Testing Messaging architecture in PHP
---

# Testing Messaging

Testing `Message Driven Architecture` can easily become nightmare from the perspective of speed and maintenance of such tests. It's really important to write easy to understand, quick and reliable test scenarios. especially in long term projects. \
That's why Ecotone comes with test supporting tools, which helps writing tests that are close to the way code runs in production, yet kept simple and isolated.

## Ecotone Lite

`Ecotone Lite` is a way to run Ecotone Application with full possibility to customize it.\
You may point exactly what classes you would like to run, turn on and off modules that you want to include or exclude.

This makes Ecotone Lite great solution for running your tests in a way that they are really close to the way your code works on production with isolation to the set of classes you would like to test.

## Configuring Ecotone Lite for your tests

```php
$ecotoneLite = EcotoneLite::bootstrapFlowTesting(
    // 1. Classes to resolve
    [User::class],
    // 2. Available services, you may inject container instead
    [new EmailConverter(), new PhoneNumberConverter(), new UuidConverter()],
    // 3. Service Configuration
    ServiceConfiguration::createWithDefaults()
        // 4. resolve all classes from given namespace
        ->withNamespaces(["App\Testing\Infrastructure\Converter"])
        // 5 add extension objects
        ->withExtensionObjects([
            // 6. register in memory repository for User
            InMemoryStateStoredRepositoryBuilder::createForAllAggregates()
        ])
        // 7. Turn off given Ecotone's modules
        ->withSkippedModulePackageNames([
            // 8. Turning off asynchronous package
            ModulePackageList::ASYNCHRONOUS_PACKAGE
        ])
);
```

1. **Classes to resolve** - The first parameter, sets list of `classes that you would like to include in this test`. Those are classes that containing Ecotone's attributes.
2. **Available services** - Provides `references to the service classes` used in this test. You may inject `PSR compatible container` instead.
3. **Service Configuration** - In here you may customize your configuration for this test case. All configs can be [find here](../../modules/ecotone-lite/#serviceconfiguration).
4. **Service Configuration -withNamespaces** - One of the configuration worth mention is `withNamespaces` this allows to use define given or set of namespaces, from which all classes will be included in this test.
5. **Service Configuration -withExtensionObjects** - This allows for defining extension object for customizing Ecotone's modules
6. **Register in memory repository -** This example extension object registers In Memory Repository for all your [state stored aggregates](../command-handling/state-stored-aggregate/)
7. **Turn off Modules -** This allows you to turn off given Ecotone's module for this test.
8. **Turning off asynchronous package -** Turning off asynchronous package for example, will make your code execute synchronously, so you won't need to bother with running consumers in your test.

{% hint style="info" %}
If you want to read more about Ecotone Lite configuration, check [Module Page](../../modules/ecotone-lite/).
{% endhint %}

## Calling Command Bus

Suppose we have user registration `Command Handler:`

```php
class UserService
{
    #[CommandHandler]
    public function register(RegisterUser $command, UserRepository $userRepository)
    {
        $userRepository->save(User::create($command->name));
    }
}
```

and we want to fetch, if `User` was stored in the repository after sending an `Command`.

<pre class="language-php"><code class="lang-php">$userRepository = new InMemoryUserRepository;

$ecotoneLite = EcotoneLite::bootstrapFlowTesting(
    // We provide list of classes which are using Ecotone's attributes
    [UserService::class],
    // We provide Service used by the Command Handler
    [UserRepository::class => $userRepository]
);
<strong>
</strong><strong>// No users in repository before calling command
</strong>$this->assertEmpty($userRepository->getAll());
<strong>
</strong><strong>$ecotoneLite->sendCommand(new RegisterUser(
</strong>    Uuid::uuid4(),
    "johny",
    Email::create("test@wp.pl"),
    PhoneNumber::create("148518518518"))
));

// User should be stored in repository
$this->assertNotEmpty($userRepository->getAll());
</code></pre>

The same way we send Command using `sendCommand,` we may send Queries - `sendQuery` and publish Events - `publishEvent.`

## Calling Event Bus

We may use `EventBus` or publish event directly using `publishEvent` support:

```php
$ecotoneLite->publishEvent(new OrderWasPlaced(Uuid::uuid4()->toString());
```

This is useful when we are testing Event Handlers directly.

```php
#[EventHandler]
public function whenOrderWasCancelled(OrderWasPlaced $event): void
{
    // something happens
}
```

## Verifying Published Events

After sending `Command`, you may verify, if given set of `events` were published as a result.&#x20;

For this Ecotone introduce `Message Collector` which intercept your message flow.

```php
$this->assertEquals(
    [new OrderWasPlaced($orderId)], 
    $ecotoneLite->getRecordedEvents()
);
```

{% hint style="success" %}
Ecotone intercept all interactions with Event/Command/Query Buses. This allows you to spy on all recorded messages, to verify their state.&#x20;
{% endhint %}

## Verifying Message Headers

If you want to validate, if Event was sent with given set of headers:

```php
$this->assertEquals(
    $executorId, 
    $ecotoneLite->getRecordedEventMessages()[0]->getHeaders()->get('executorId')
);
```

## **Verifying Commands/Queries**

You may also verify, if `Command` or `Query` was sent:

```php
    /**
     * @return array<int, mixed>
     */
    public function getRecordedCommands(): array;

    /**
     *  Allows to assert metadata of the message
     *
     * @return Message[]
     */
    public function getRecordedCommandMessages(): array;

    /**
     * @return array<int, mixed>
     */
    public function getRecordedQueries(): array;

    /**
     *  Allows to assert metadata of the message
     *
     * @return Message[]
     */
    public function getRecordedQueryMessages(): array;
    
    /**
     * @return array<int, mixed>
     */
    public function getRecordedEvents(): array
```

### Discarding recorded messages

In case you're not interested in current messages, you may clean up `Message Collector`

```php
$ecotoneLite->discardRecordedMessages();
```

## Caching Configuration

Ecotone Lite tests are quick to run as the boot minimal version of Ecotone which is supposed to handle given set of classes. This way we avoid booting whole Application in order to run tests that cover specific scenario.

The only part which add extra milliseconds to the tests execution, is bootstrapping the configuration. However Ecotone does cache it between the test runs and mark it as stale when related test files does change. This means we may have great volume of tests using Ecotone Lite, and execution speed will be preserved.

{% hint style="success" %}
By default if not changed Ecotone Lite, will store the config in temporary folder under ecotone catalog: "/tmp/ecotone".\
Ecotone will reload cache on configuration changes, yet we may always remove the catalog manually in case.
{% endhint %}
