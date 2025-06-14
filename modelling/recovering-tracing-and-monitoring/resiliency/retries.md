# Retries

## Instant Retries

Instant Retries are powerful self-healing mechanism, which helps Application to automatically recover from failures.&#x20;

The are especially useful to handle temporary issues, like optimistic locking, momentary unavailability of the external service which we want to call or database connection failures. This way we can recover without affecting our end users without any effort on the Developer side.

Instant retries can be enabled for CommandBus and for Asynchronous Processing.

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

This will retry instantly when your message is handled asynchronously. This applies to Command and Events. Take under consideration that Ecotone [isolates handling asynchronous events](broken-reference), so it's safe to retry them.

{% hint style="success" %}
By using instant retries for asynchronous endpoints we keep message ordering.&#x20;
{% endhint %}

## Customized Instant Retries

The `InstantRetry` attribute allows you to specify different strategies for Retry in order to be able to customize it for specific Business use cases. For example we may create new Command Bus which will retry on **NetworkException** then use that in specific cases with custom retry.\
We do it by extending **CommandBus** interface and adding **InstantRetry** attribute.

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
