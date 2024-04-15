---
description: Saga / Process Manager - Stateful Workflow PHP
---

# Stateful Workflows - Saga

Not all Workflows need to maintain state. In case of step by step Workflows we mostly can handle them using stateless input / output functionality. However there may a cases, where we would like to maintain the state and the reasoning behind that may come from:

* Our Workflow branches into multiple separate flows, which need to be combined back to make the decision
* Our Workflow involves manual steps, like approving / declining before Workflow can resume or make the decision
* Our Workflow is long running process, which can take hours or days and we would like to have high visibility of the current state

For above cases when Stateful Workflow is needed, we can make use of [Sagas](../../modelling/command-handling/saga.md).

## Saga - Stateful Workflow

Saga is a Stateful Workflow, as it's running on the same basics as Stateless Workflows using Input and Output Channels under the hood. The difference is that it does persist the state, therefore the decisions it makes can be based on previous executions.

We define Sagas using Saga Attribute and by defining Identifier.&#x20;

```php
#[Saga]
final class OrderProcess
{
    private function __construct(
        // Saga has to have Identifier, so we can fetch it from the Storage
        #[Identifier] private string $orderId,
        private OrderProcessStatus   $orderStatus,
    ) {}

    // This our factory method, which provides new Saga instance when Order Was Placed
    #[EventHandler]
    public static function startWhen(OrderWasPlaced $event): self
    {
        return new self($event->orderId, OrderProcessStatus::PLACED);
    }
}
```

Sagas are initialized using Factory Methods (Methods marked as static).  We can initialize Saga by subscribing to an Event, or by trigger an Command Action.

To store Saga we will be using [Repositories](../../modelling/command-handling/business-interface/repository.md). We can provide custom implementation or use inbuilt repositories like Doctrine ORM, Eloquent Model or Ecotone's Document Store.

{% hint style="success" %}
Aggregates and Sagas provides the same functionality, both can subscribe to Events and receive Commands, and both are persisted using Repositories. Therefore whatever we use one or another, is about making the code more explicit. As Sagas are more meant for business processes, and Aggregate for protecting business invariants.
{% endhint %}

## Subscribing to Events and taking action

We could have Ship Order Command Handler, which we would like to trigger from Saga:

```php
class Shipment
{
    #[CommandHandler("shipOrder")]
    public function prepareForShipment(ShipOrder $event): void
    {
        // ship the Order
    }
}
```

Saga can subscribe to Event and take action on that to send this Command:

```php
#[Saga]
final class OrderProcess
{   
    (...)

    // Message will be delivered to Message Channel "shipOrder"
    #[EventHandler(outputChannelName: "shipOrder")]
    public function prepareForShipment(PaymentWasSuccessful $event): ShipOrder
    {
        $this->orderStatus = OrderProcessStatus::READY_TO_BE_SHIPPED;
        
        return new ShipOrder($this->orderId);
    }
```

In above example we change Saga's state and then return an Message to be delivered to the Output Channel just like we did for [Stateless Workflows](stateful-workflows-saga.md).

We could also take action by sending an Message using Command Bus.

```php
#[EventHandler(outputChannelName: "shipOrder")]
public function prepareForShipment(PaymentWasSuccessful $event, CommandBus $commandBus): void
{
    $this->orderStatus = OrderProcessStatus::READY_TO_BE_SHIPPED;
    
    $commandBus->send(new ShipOrder($this->orderId));
}
```

{% hint style="success" %}
The difference between using **Output Channel** and **Command Bus** is in matter of time execution. \
\- When we use **Command Bus**, Message will be send before Saga state will be persisted.\
\- When we will use **Output Channel**, Saga state will first persisted and then Message will be send.
{% endhint %}

## Publishing Events from Saga

We can publish Events from Saga, to trigger next actions. This can be useful for example, if we would like to trigger time based actions after X amount after Saga was started. \
\
For example we may want to close Order, if it was not paid after 24 hours.

```php
#[Saga]
final class OrderProcess
{
    use WithEvents;

    private function __construct(
        #[Identifier] private string $orderId,
        private OrderProcessStatus   $orderStatus,
        private bool $isPaid,
    ) {
        $this->recordThat(new OrderProcessWasStarted($this->orderId));
    }

    #[Delayed(new TimeSpan(days: 1))]
    #[Asynchronous('async')]
    #[EventHandler]
    public function startWhen(OrderProcessWasStarted $event): void
    {
        if (!$this-isPaid()) {
            $this->orderStatus = OrderProcessStatus::CANCELLED;
        }
    }
    
    (...)
}
```

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

{% hint style="success" %}
This option can also be used together with Command Handlers.
{% endhint %}

## Fetching State from Saga

We may want to fetch the State from Saga. This may be useful for example to provide to the Customer current state at which we are at:

```php
#[Saga]
final class OrderProcess
{   
    (...)

    #[QueryHandler("orderProcess.getStatus")]
    public function getStatus(): OrderProcessStatus
    {
        return $this->orderStatus;
    }
```

Then we can fetch the state using Query Bus

```php
public function OrderController
{
    public function __construct(private QueryBus $queryBus) {}
    
    public function getStatus(Request $request): Response
    {
        $orderId = $request->get('orderId');
        
        return new Response([
            'status' => $this->queryBus->sendWithRouting(
                'orderProcess.getStatus',
                metadata: [
                    // aggregate.id is special metadata, that can be used for both Sagas and Aggregates
                    'aggregate.id' => $orderId
                ]
            )
        ]);
    }
}
```

## Unordered Events

We may actually be unsure about ordering of some Events. It may happen that same Event may to come us at different stages of the Saga. So it will either initialize the Saga or call the Action on Saga. \
\
To solve this we Ecotone allows for Method Redirection based on Saga's existance:

```php
#[Saga] 
class OrderFulfillment
{
    private function __construct(#[Identifier] private string $orderId)
    {
    }

    #[EventHandler]
    public static function startByPlacedOrder(OrderWasPlacedEvent $event) : self
    {
        // initialize saga
    }

    #[EventHandler]
    public function whenOrderWasPlaced(OrderWasPlacedEvent $event) : void
    {
        // handle action
    }
    
    (...)
}
```

In the above example, we used the **same Event for Factory and Actions Method.** \
Ecotone if given Saga does not exists yet, will call factory method, otherwise Action.

```php
# Factory Method will be called if Saga does not exists
#[EventHandler("whenOrderWasPlaced")]
public static function startByPlacedOrder(OrderWasPlacedEvent $event) : self

# Action method will be called if Saga already exists
#[EventHandler("whenOrderWasPlaced")]
public function whenOrderWasPlaced(OrderWasPlacedEvent $event, CommandBus $commandBus) : self
```

This solution will prevent us from depending on the order of events, without introducing any if statements or routing functionality in our business code.

## Targeting Identifier from Event/Command

Whenever Event or Command comes in to Saga, we need to correlate it with given Saga's instance. \
For this we can leverage Ecotone's support for [Identifier Mapping](../../modelling/command-handling/identifier-mapping.md). \
This will give us ability to map Saga using different possibilites.
