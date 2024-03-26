# Retries

## Instant Retries

Ecotone provides instant retries, which triggers automatically, if given Message failed and tries to recover immediately.

Instant retries may be useful in case of temporary issues, like optimistic locking, momentary unavailability of the external service which we want to call or database connection failure.&#x20;

### Command Bus Instant Retries

In order to set up instant retries for Command Bus, you [Service Context](../../../messaging/service-application-configuration.md) configuration.

```php
#[ServiceContext]
public function registerRetries()
{
    return InstantRetryConfiguration::createWithDefaults()
             ->withCommandBusRetry(
                      true, // is enabled
                      3, // max retries
                      [DatabaseConnectionFailure::class, OptimisticLockingException::class] // list of exceptions to be retried, leave empty if all should be retried
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
                      true, // is enabled
                      3, // max retries
                      [DatabaseConnectionFailure::class, OptimisticLockingException::class] // list of exceptions to be retried, leave empty if all should be retried
             )
}
```

This will retry instantly when your message is handled asynchronously. This applies to Command and Events. Take under consideration that Ecotone [isolates handling asynchronous events](broken-reference), so it's safe to retry them.

{% hint style="success" %}
By using instant retries for asynchronous endpoints we keep message ordering.&#x20;
{% endhint %}

## Delayed Retries

Delayed retries are helpful in case, we can't recover instantly. This may happen for example due longer downtime of external service, which we integrate with. \
In those situations we may try to self-heal the application, by delaying failed Message for given period of time. This way we can retry the call to given service after some time, and if everything is fine, then we will successfully handle the message.

### Installation

First [Error Channel](error-channel-and-dead-letter.md#error-channel) need to be set up for your Application, then you may configure retries.

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

## Safe Retries

There is crucial difference between Ecotone and other PHP Frameworks in a way it enables safe retries.

Let's consider asynchronous scenario, where we want send order confirmation and reserve products in Stock via HTTP call, when Order Was Placed. This could potentially look like this:

```php
#[Asynchronous("asynchronous_messages")]
#[EventHandler(endpointId: "notifyAboutNewOrder")]
public function notifyAboutNewOrder(OrderWasPlaced $event, NotificationService $notificationService) : void
{
    $notificationService->notifyAboutNewOrder($event->getOrderId());
}

#[Asynchronous("asynchronous_messages")]
#[EventHandler(endpointId: "reserveItemsInStock")]
public function reserveItemsInStock(OrderWasPlaced $event, StockClient $stockClient): void
{
    $stockClient->reserve($event->getOrderId(), $event->getProducts());
}
```

Now imagine that sending to Stock fails and we want to retry. If we would retry whole Event, we would retry "notifyAboutNewOrder" method, this would lead to sending an notification twice. It's easy to imagine scenarios where this could lead to even worse situations, where side effect could lead to double booking, trigger an second payment etc. \
That's is why Ecotone implements Safe Retries.

### Sending a copy to each of the Handlers

In Ecotone each of the Handlers will receive it's own copy of the Event and will handle it in full isolation.

This means that under the hood, there would be two messages sent to `asynchronous_messages` \
each targeting specific Event Handler.\
This bring safety to retrying events, as in case of failure, we will only retry the Handler that actually failed.

{% hint style="info" %}
As in Ecotone it's the Handler that becomes Asynchronous (not Event itself) you may customize the behaviour to your needs.\
If you want, you may:&#x20;

* Run one Event Handler synchronously and the other asynchronously.&#x20;
* You may decide to use different Message Channels for each of the Asynchronous Event Handlers.
* You delay or add priority to  one Handler and to the other not&#x20;
{% endhint %}
