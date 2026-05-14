---
description: Configuring automatic message retry strategies in Ecotone PHP
---

# Retries

## Instant Retries

Your handler hits a transient DB connection drop, an optimistic-lock collision, or a 503 from Stripe. Retrying once usually fixes it. Retrying ten times in a busy loop hammers your DB. **Instant Retry** is the bounded, exception-filtered retry — same call stack, no broker round-trip — that handles 90% of transient failures in production without bothering the user.

It is especially useful for optimistic locking, momentary unavailability of an external service, and database connection failures, where the right answer is "try again immediately." Instant retries can be enabled for the CommandBus and for Asynchronous Processing.

## Global Instant Retries

### Command Bus instant retries

In order to set up instant retries for Command Bus, you [Service Context](../../../messaging/service-application-configuration.md) configuration.

```php
#[ServiceContext]
public function registerRetries()
{
    return InstantRetryConfiguration::createWithDefaults()
             ->withCommandBusRetry(
                      isEnabled: true,
                      retryTimes: 3, // max retries
                      retryExceptions: [DatabaseConnectionFailure::class, OptimisticLockingException::class] // list of exceptions to be retried, leave empty if all should be retried
             )
}
```

This will retry your `synchronous Command Handlers`.

### Asynchronous Instant Retries

```php
#[ServiceContext]
public function registerRetries()
{
    return InstantRetryConfiguration::createWithDefaults()
             ->withAsynchronousEndpointsRetry(
                      isEnabled: true,
                      retryTimes: 3, // max retries
                      retryExceptions: [DatabaseConnectionFailure::class, OptimisticLockingException::class] // list of exceptions to be retried, leave empty if all should be retried
             )
}
```

This will retry instantly when your message is handled asynchronously. This applies to Command and Events. Take under consideration that Ecotone [isolates handling asynchronous events](../message-handling-isolation.md), so it's safe to retry them.

{% hint style="success" %}
By using instant retries for asynchronous endpoints we keep message ordering.&#x20;
{% endhint %}

## Command Bus Instant Retries

Create custom Command Buses with tailored retry strategies for specific business scenarios. Instead of scattering try/catch retry loops across handlers, declare retry behaviour as an attribute -- specify which exceptions to retry and how many times.

**You'll know you need this when:**

* Database deadlocks cause intermittent command handler failures
* External API calls fail transiently and a simple retry would succeed
* You have try/catch retry loops scattered across your handlers
* High-concurrency scenarios produce optimistic locking collisions that resolve on retry

{% hint style="success" %}
Customized Instant Retries are available as part of **Ecotone Enterprise.**
{% endhint %}

### Instant retries times

To set up Customized Instant Retries, we will extend **CommandBus** and provide the attribute

```php
#[InstantRetry(retryTimes: 2)]
interface ReliableCommandBus extends CommandBus {}
```

**CommandBusWithRetry** will be automatically registered in our Dependency Container and available for use.&#x20;

Now whenever we will send an Command using this specific Command Bus, it will do two extra retries:

```php
$this->commandBusWithRetry->send(new RegisterNewUser());
```

### Instant Retries exceptions

The same way we can define specific exception list which should be retried for our customized Command Bus:

```php
#[InstantRetry(retryTimes: 2, exceptions: [NetworkException::class])]
interface ReliableCommandBus extends CommandBus {}
```

Only the exceptions defined in exception list will be retried.

## Asynchronous Delayed Retries

Delayed retries are helpful in case, we can't recover instantly. This may happen for example due longer downtime of external service, which we integrate with. \
In those situations we may try to self-heal the application, by delaying failed Message for given period of time. This way we can retry the call to given service after some time, and if everything is fine, then we will successfully handle the message.

Ecotone resends the Message to original channel with delay. This way we don't block processing during awaiting time, and we can continue consuming next messages. When Message will be ready (after delay time), it will be picked up from the Queue.

### Installation

First [Error Channel](error-channel-and-dead-letter/#error-channel) need to be set up for your Application, then you may configure retries.

### Using Default Delayed Retry Strategy

If you want to use inbuilt Error Retry Strategy and set retry attempts, backoff strategy, initial delay etc, you may configure using `ErrorHandlerConfiguration` from [ServiceContext](../../../messaging/service-application-configuration.md).

```php
#[ServiceContext]
public function errorConfiguration()
{
    return ErrorHandlerConfiguration::create(
        "errorChannel",
        RetryTemplateBuilder::exponentialBackoff(1000, 10)
            ->maxRetryAttempts(3)
    );
}
```

## Using Custom Delayed Strategy for Consumer

```php
#[Asynchronous("asynchronous_messages")]
#[EventHandler(endpointId: "notifyAboutNewOrder")]
public function notifyAboutNewOrder(OrderWasPlaced $event, NotificationService $notificationService) : void
{
    $notificationService->notifyAboutNewOrder($event->getOrderId());
}
```

When we have consumer named **"asynchronous\_messages"**, then we can define PollingMetadata with customer error Channel.

```php
#[ServiceContext]
public function errorConfiguration()
{
    return PollingMetadata::create("asynchronous_messages")
            ->setErrorChannel("customErrorChannel");
}

#[ServiceContext]
public function errorConfiguration()
{
    return ErrorHandlerConfiguration::create(
        "customErrorChannel",
        RetryTemplateBuilder::exponentialBackoff(100, 3)
            ->maxRetryAttempts(2)
    );
}
```
