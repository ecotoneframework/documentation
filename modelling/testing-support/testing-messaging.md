---
description: Testing Messaging architecture in PHP
---

# Testing Messaging

Testing `Message Driven Architecture` can easily become a nightmare from the perspective of speed and maintenance of such tests. It's really important to write easy to understand, quick and reliable test scenarios, especially in long-term projects. \
That's why Ecotone comes with test supporting tools, which help write tests that are close to the way code runs in production, yet kept simple and isolated.

## Ecotone Lite

When you need to test your message-driven code in isolation without booting your entire application, Ecotone Lite provides a lightweight solution. It allows you to run a minimal version of Ecotone with full customization capabilities.\
You can specify exactly which classes to include, and enable or disable specific modules as needed for your test scenario.

This makes Ecotone Lite an excellent solution for writing tests that are close to production behavior while maintaining fast execution and proper isolation. Instead of loading your entire application with all its dependencies, you only load what's necessary for the specific functionality you're testing.

## Configuring Ecotone Lite for your tests

Here's how to set up Ecotone Lite for testing. Each parameter serves a specific purpose in creating an isolated test environment:

```php
$ecotoneLite = EcotoneLite::bootstrapFlowTesting(
    // 1. Classes to resolve - specify which classes contain Ecotone attributes
    [User::class],
    // 2. Available services - provide service instances or inject a container
    [new EmailConverter(), new PhoneNumberConverter(), new UuidConverter()],
    // 3. Service Configuration - customize the test environment
    ServiceConfiguration::createWithDefaults()
        // 4. Load classes from specific namespaces automatically
        ->withNamespaces(["App\Testing\Infrastructure\Converter"])
        // 5. Add extension objects for additional functionality
        ->withExtensionObjects([
            // 6. Use in-memory repositories for fast testing
            InMemoryStateStoredRepositoryBuilder::createForAllAggregates()
        ])
        // 7. Disable modules not needed for this test
        ->withSkippedModulePackageNames([
            // 8. Skip async processing to run tests synchronously
            ModulePackageList::ASYNCHRONOUS_PACKAGE
        ])
);
```

**Understanding the Configuration Parameters:**

1. **Classes to resolve** - Specify which classes contain Ecotone attributes (like `#[CommandHandler]`, `#[EventHandler]`, etc.) that you want to test. Only include the classes relevant to your test scenario.

2. **Available services** - Provide instances of services your handlers need, or inject a PSR-compatible container. This replaces your normal dependency injection setup for testing.

3. **Service Configuration** - Customize how Ecotone behaves in your test. All available configurations are documented [here](../../modules/ecotone-lite/#serviceconfiguration).

4. **withNamespaces** - Instead of listing individual classes, you can tell Ecotone to automatically discover all classes with attributes in specific namespaces. This is useful when you have many related classes.

5. **withExtensionObjects** - Add special objects that modify how Ecotone works. For example, you can replace real databases with in-memory alternatives for faster testing.

6. **In-memory repositories** - This extension replaces your normal database repositories with fast in-memory versions, perfect for unit and integration tests.

7. **Skipping modules** - Disable Ecotone modules you don't need for your specific test, making tests faster and more focused.

8. **Synchronous execution** - By disabling the asynchronous package, messages that would normally be processed asynchronously (like queued events) run immediately in your test, making assertions easier.

{% hint style="info" %}
**Related Documentation:**
- [Ecotone Lite Module](../../modules/ecotone-lite/) - Complete configuration reference
- [Testing Aggregates and Sagas](testing-aggregates-and-sagas-with-message-flows.md) - Advanced testing patterns
- [Testing Event Sourcing](testing-event-sourcing-applications.md) - Event sourcing specific testing
- [Testing Asynchronous Messaging](testing-asynchronous-messaging.md) - Async testing strategies
{% endhint %}

## Testing Command Handlers

Let's walk through a practical example. Suppose we have a user registration Command Handler that saves users to a repository:

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

We want to test that when we send a `RegisterUser` command, the user is actually stored in the repository. Here's how to test this behavior:

```php
// 1. Create an in-memory repository for testing
$userRepository = new InMemoryUserRepository;

$ecotoneLite = EcotoneLite::bootstrapFlowTesting(
    // 2. Tell Ecotone which classes contain handlers to test
    [UserService::class],
    // 3. Provide the repository instance our handler needs
    [UserRepository::class => $userRepository]
);

// 4. Verify repository is empty before we start
$this->assertEmpty($userRepository->getAll());

// 5. Send the command through Ecotone's command bus
$ecotoneLite->sendCommand(new RegisterUser(
    Uuid::uuid4(),
    "johny",
    Email::create("test@wp.pl"),
    PhoneNumber::create("148518518518")
));

// 6. Verify the command handler did its job
$this->assertNotEmpty($userRepository->getAll());
```

**What's happening here:**
1. We create an in-memory repository that behaves like a real one but stores data in memory for fast testing
2. We tell Ecotone to load our `UserService` class which contains the Command Handler
3. We provide the repository instance that our handler will receive when called
4. We verify our starting state (empty repository)
5. We send the command using Ecotone's testing API - this will trigger our handler
6. We verify the expected outcome (user was saved)

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

When testing command handlers that publish events, you need to verify that the correct events were published as a result of your command. This is crucial for ensuring your business logic triggers the right side effects.

Ecotone provides a `Message Collector` that automatically captures all messages flowing through your system during testing, allowing you to inspect what events were published:

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

Sometimes you need to verify not just what events were published, but also what metadata (headers) they carried. This is important when your business logic depends on contextual information like user IDs, timestamps, or correlation IDs:

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

One concern when using testing frameworks is performance - you don't want your test suite to become slow as it grows. Ecotone Lite addresses this by running only the minimal parts of Ecotone needed for your specific test, avoiding the overhead of booting your entire application.

The main performance consideration is the initial configuration bootstrapping. To solve this, Ecotone automatically caches the configuration between test runs and intelligently invalidates the cache when your test files change. This means you can have hundreds of Ecotone Lite tests while maintaining fast execution times.

{% hint style="success" %}
By default if not changed, Ecotone Lite will store the config in temporary folder under ecotone catalog: "/tmp/ecotone".\
Ecotone will reload cache on configuration changes, yet we may always remove the catalog manually in case.
{% endhint %}

## Common Pitfalls and Troubleshooting

### Missing Dependencies in Test Setup

**Problem:** Your test fails with "Service not found" or similar dependency injection errors.

**Solution:** Make sure you provide all services that your handlers need in the second parameter of `bootstrapFlowTesting()`:

```php
$ecotoneLite = EcotoneLite::bootstrapFlowTesting(
    [UserService::class],
    [
        UserRepository::class => new InMemoryUserRepository(),
        EmailService::class => new MockEmailService(),
        // Add all services your handlers depend on
    ]
);
```

### Asynchronous Code Not Working in Tests

**Problem:** Your asynchronous event handlers or command handlers don't seem to execute in tests.

**Solution:** By default, Ecotone Lite runs everything synchronously for testing. If you need to test asynchronous behavior, don't skip the asynchronous package:

```php
// Add this line if you want to test async behavior
// ->withSkippedModulePackageNames([ModulePackageList::ASYNCHRONOUS_PACKAGE])
```

### Events Not Being Recorded

**Problem:** `getRecordedEvents()` returns empty array even though you expect events.

**Solution:**
1. Make sure your command handler actually publishes events
2. Check that you're calling `getRecordedEvents()` on the same test support instance
3. Verify your event handlers are properly annotated with `#[EventHandler]`

### Class Not Found Errors

**Problem:** Ecotone can't find your classes during testing.

**Solution:** Either include the class in the first parameter or use `withNamespaces()`:

```php
// Option 1: List classes explicitly
$ecotoneLite = EcotoneLite::bootstrapFlowTesting([UserService::class, OrderService::class]);

// Option 2: Use namespaces
$ecotoneLite = EcotoneLite::bootstrapFlowTesting(
    [],
    [],
    ServiceConfiguration::createWithDefaults()->withNamespaces(['App\\Service'])
);
```
