---
description: Process Manager Saga PHP
---

# Saga Introduction

Read [Aggregate Introduction](state-stored-aggregate/) sections first to get wider context before diving into Sagas.

## Handling Sagas

**Saga** is responsible for coordination of long running processes. It can store information about what happened and make a decision what to do next in result.

```php
#[Saga]
class OrderFulfillment
{
    #[Identifier] 
    private string $orderId;

    private bool $isFinished;
   
    private function __construct(string $orderId)
    {
        $this->orderId = $orderId;
    }

    // Static Factory Method. Create new instance of Saga.
    #[EventHandler] 
    public static function start(OrderWasPlacedEvent $event) : self
    {
        return new self($event->getOrderId());
    }

    // Action method in existing Saga instance
    #[EventHandler] 
    public function whenPaymentWasDone(PaymentWasFinishedEvent $event, CommandBus $commandBus) : self 
    {
       if ($this->isFinished) {
           return;
       }
    
        $this->isFinished = true;
        $commandBus->send(new ShipOrderCommand($this->orderId));
    }
}
```

To enable given Class as Saga, we mark it with **Saga** or **EventSourcingSaga** Attribute. This works in the same way as stated-stored [Aggregate](state-stored-aggregate/) or [Event Sourced Aggregate](broken-reference).

In order to trigger the saga, whenever given Event happens, we use **EventHandler** Attribute. It works like normal Event Handler, yet it will load the Saga and the save it, just like it would be done with Aggregates.

In above example we the "**start" method** is factory method for Saga. It will create new instance of given Saga, whenever specific Event happen.\
On other hand **"whenPaymentWasDone" method** is action method, which expects Saga to be already initialized.

## Storing Saga's State

Saga just as Aggregates are stored using [Repository](repository.md) implementation, you may store Saga as event stream or use only current state using Standard Repositories.

## Targeting Identifier from Event/Command

As Saga is identified by identifier just like an Aggregate, events that trigger actions on the Saga needs to be correlated with specific instance.\
When we do have event like `PaymentWasFinishedEvent` we need to tell `Ecotone` which instance of `OrderFulfillment` it should be retrieve from [Repository](repository.md) to call method on.

This is done automatically, when property name in `Event` is the same as property marked as       `#[Identifier]` in Saga.&#x20;

```php
#[Saga]
class OrderFulfillment
{
    #[Identifier] 
    private string $orderId;
```

Then if Event contains of orderId, the mapping will be done automatically:

```php
class PaymentWasFinishedEvent
{
    private string $orderId;
}
```

If the property name is different we need to give `Ecotone` a hint, how to correlate identifiers.&#x20;

```php
class SomeEvent
{
    #[TargetIdentifier("orderId")] 
    private string $purchaseId;
}
```

## Targeting Identifier from Metadata

When there is no property to correlate inside `Command` or `Event`, we can make use of `Before` or `Presend` [Interceptors](../extending-messaging-middlewares/interceptors.md) to enrich event's metadata with required identifier. \
When we've the identifier inside `Metadata` then we can use `identifierMetadataMapping.`\
\
Suppose the `orderId` identifier is available in metadata under key `orderNumber`, then we can tell Message Handler to use this mapping:

```php
#[EventHandler(identifierMetadataMapping: ["orderId" => "orderNumber"])]
public function failPayment(PaymentWasFailedEvent $event, CommandBus $commandBus) : self 
{
   // do something with $event
}
```

## Unordered Events

In the [starting example](saga.md#handling-saga) we have assumed, that the first event we will receive is `OrderWasPlacedEvent` which will create new Saga instance, and the second will call action on the existing Saga `(PaymentWasFinishedEvent).` \
This kind of assumption are not always possible, especially when we are subscribing to Events from [Different Systems](../microservices-php/distributed-bus.md#consuming-distributed-events).

\
To solve this, Ecotone allows to build Sagas in a way, that order of the Messages does not really matter:

```php
#[Saga] 
class OrderFulfillment
{
    #[Identifier] 
    private string $orderId;

    private bool $isFinished;

    private function __construct(string $orderId)
    {
        $this->orderId = $orderId;
    }

    #[EventHandler]
    public static function startByPlacedOrder(OrderWasPlacedEvent $event) : self
    {
        return new self($event->getOrderId());
    }

    // Make factory method for PaymentWasFinish
    #[EventHandler]
    public static function startByPaymentFinished(PaymentWasFinished $event) : self 
    {
        return new self($event->getOrderId());
    }

    // Make action method for OrderWasPlaced
    #[EventHandler]
    public function whenOrderWasPlaced(OrderWasPlacedEvent $event, CommandBus $commandBus) : self
    {
        if ($this->isFinished) {
            return;
        }

        $this->isFinished = true;
        $commandBus->send(new ShipOrderCommand($this->orderId));
    }
    
    #[EventHandler] 
    public function whenPaymentWasDone(PaymentWasFinishedEvent $event, CommandBus $commandBus) : self
    {
        if ($this->isFinished) {
            return;
        }

        $this->isFinished = true;
        $commandBus->send(new ShipOrderCommand($this->orderId));
    }
}
```

In the above example, we used the same Events for Factory and Actions Method. \
Ecotone if given Saga does not exists yet, will call factory method, otherwise Action.

```php
#[EventHandler("whenOrderWasPlaced")]
public static function startByPlacedOrder(OrderWasPlacedEvent $event) : self

#[EventHandler("whenOrderWasPlaced")]
public function whenOrderWasPlaced(OrderWasPlacedEvent $event, CommandBus $commandBus) : self
```

\
This solution will prevent us from depending on the order of events, without introducing any if statements or routing functionality in our business code.

## Ignoring Events

There may be situations, when we will want to handle events only if Saga already started. \
Suppose we want to send promotion code to the customer, if he received great customer badge first, otherwise we want to skip.

```php
#[Saga] 
class OrderFulfillment
{
    #[Identifier] 
    private string $customerId;

    private function __construct(string $customerId)
    {
        $this->customerId = $customerId;
    }

    #[EventHandler] 
    public static function start(ReceivedGreatCustomerBadge $event) : void
    {
        return new self($event->getCustomerId());
    }

    #[EventHandler(dropMessageOnNotFound: true)]
    public function whenNewOrderWasPlaced(OrderWasPlaced $event, CommandBus $commandBus) : void 
    {
        $commandBus->send(new PromotionCode($this->customerId));
    }
}
```

&#x20;We filter the Event by adding `dropMessageOnNotFound.`

```php
EventHandler(dropMessageOnNotFound=true)
```

If this saga instance will be not found, then this event will be dropped and our `whenNewOrderWasPlaced` method will not be called.

{% hint style="info" %}
Options we used in here, can also be applied to Command Handlers
{% endhint %}

## Handling Commands

In case of Ecotone we may trigger Sagas using Command too.\
This is especially useful when some flows needs to be manually kicked off, or require some acceptance steps along the way.

We may have for example Verification process, which starts by Customer being registered, yet require manual confirmation is Customer Credit Profile is trustworthy.

```php
#[Saga] 
class CustomerVerificationProcess
{
    #[Identifier] 
    private string $customerId;
    private bool $creditProfileAccepted = false;

    private function __construct(string $customerId)
    {
        $this->customerId = $customerId;
    }

    #[EventHandler] 
    public static function start(CustomerRegistered $event) : self
    {
        return new self($event->getCustomerId());
    }
    
    #[CommandHandler]
    public function acceptCreditProfile(AcceptCreditProfile $command) : void 
    {
        $this->creditProfileAccepted = true;
    }
}
```

## Time based Actions

There may be situations when we will want to do time based actions. \
We may want to give opportunity to receive the promotion code for 24 hours after Customer have registered, yet only if he has verified his email during that time.

To make it happen we will use [Asynchronous Processing](../asynchronous-handling/), which allows for delaying of Messages.

```php
#[Saga] 
class PromotionCodeSaga
{
    // We've added possibility to record events for this Saga.
    use WithEvents;

    #[Identifier] 
    private string $customerId;
    private bool $isFinished;

    private function __construct(string $customerId)
    {
        $this->customerId = $customerId;
        $this->isFinished = false;
        
        $this->recordThat(new PromotionCodeWasStarted($this->customerId));
    }

    #[EventHandler] 
    public static function start(CustomerRegistered $event) : void
    {
        return new self($command->getCustomerId());
    }

    // Time in ms, this method will execute 24 hours after promotion have started
    #[Delayed(1000 * 60 * 60 * 24)]
    #[Asynchronous("async_channel")]
    #[EventHandler(endpointId: "promotion_code_saga.timeout")]
    public function timeout(PromotionCodeWasStarted $event): void
    {
        $this->isFinished = true;
    }

    #[EventHandler(dropMessageOnNotFound: true)]
    public function whenNewOrderWasPlaced(
        EmailWasVerified $event, 
        CommandBus $commandBus
    ) : void 
    {
        if ($this->isFinished) {
            return;
        }
    
        $this->isFinished = true;
        $commandBus->send(new SendPromotionCode($this->customerId));
    }
}
```

Saga just like Aggregate can record new Events in case of need. And this is what we do in the constructor in the above example:

```php
private function __construct(string $customerId)
{
    $this->customerId = $customerId;
    $this->isFinished = false;
    
    $this->recordThat(new PromotionCodeWasStarted($this->customerId));
}
```

Then we can subscribe to this Event with delay of 24 hours. This way we will we can trigger actions after certain period of time without any additional configurations, keep the code explicit about what is actually happening:

```php
#[Delayed(1000 * 60 * 60 * 24)]
#[Asynchronous("async_channel")]
#[EventHandler(endpointId: "promotion_code_saga.timeout")]
public function timeout(PromotionCodeWasStarted $event): void
{
    $this->isFinished = true;
}
```

{% hint style="success" %}
Subscribing to creation events with delay, makes the complex flow effortless. \
We are using features of underlying Messaging Architecture.\
\
Delays can be used the same way inside the Aggregates too.
{% endhint %}
