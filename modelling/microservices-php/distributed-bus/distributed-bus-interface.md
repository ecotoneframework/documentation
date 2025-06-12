# Distributed Bus Interface

## Distribution Bus

Ecotone does use given Provider abstractions to create topology which can be used to route Messages from one Service to another.\
\
When given Provider is enabled as a Publisher, then **DistributedBus** interface is registered in our Dependency Container. DistributedBus is an interface entrypoint for sending Messages between Services. It does provides way to send Commands, Events and generic Messages (the intention behind the last one will be reveal soon).

```php
interface DistributedBus
{
    // Command distribution
    public function sendCommand() : void;
    public function convertAndSendCommand() : void;

    // Event distribution
    public function publishEvent() : void;
    public function convertAndPublishEvent() : void;
    
    // Generic Message distribution
    public function sendMessage(): void;    
}
```

\
For example, when we are sending command, we are stating which Service we aim to send the Message to. This way Ecotone is aware where given Message should land.&#x20;

## Command Distribution

Command have always only single Message Handler that handle specific Command. One the Service (Application) level this ensured by constraint, as no more than one Command Handler can have same routing key. \
However if we would simply publish an Command multiple Services could subscribe to it, which would defeat the intention behind Command Message.  It's much easier to reason about how the system behaves, when we can rely on Command having single point of handling, and Events having multiple ones.&#x20;

Therefore to achieve the isolation, when Command is send we are stating given Service Name to which it should land. This ensures that Message will go no where else, and then on the Service level, it's ensured by Ecotone configuration that only single Command Handler will be handling this Message.

### Sending Distributed Commands

To send Command Messages we will be using DistributedBus:

```php
interface DistributedBus
{
    public function convertAndSendCommand(
        string $targetServiceName, 
        string $routingKey, 
        object|array $command, 
        array $metadata = []
    ) : void;
    
(...)
```

* **targetServiceName** - is a [Service Name](../#configuration) of targeted Service
* **routingKey** - is a routing key under which CommandHandler is registered on targeted Service

```php
public function changeBillingDetails(DistributedBus $distributedBus)
{
    $distributedBus->sendCommand(
        targetServiceName: "billing",
        routingKey: "billing.changeDetails",
        '["personId":"123","billingDetails":"01111"]',
        "application/json"
    );
}
```

### Consuming distributed command

General flow behind consuming Distributed Command Messages is to add Distributed attribute on top of our Command Handler. This way we state the intention that this Command Handler should take a part in Command Distribution.

```php
#[Distributed]
#[CommandHandler("billing.changeDetails")]
public function changeBillingDetails(ChangeBillingDetails $command) : void
{
    // do something with billing details
}
```

{% hint style="info" %}
To expose command handler for service distribution mark it with `Distributed` attribute.
{% endhint %}

### Event Distribution

Events are different from Commands as there may be multiple Services involved into Event subscriptions. Therefore when we publish Event Message we don't state where should it land (Service Name) as from publisher side we are decoupled from that decision. \
\
Each Distributed Consumer is stating however which Event it would like to subscribe too. Therefore each Service can connect to subscription, when it becomes interested in particular distributed Event.

### Publishing Distributed Events

To send Event Messages we will be using DistributedBus:

```php
interface DistributedBus
{
    public function convertAndPublishEvent(
        string $routingKey, 
        object|array $event, 
        array $metadata
    ) : void;
    
(...)
```

* `routingKey` - is a routing key under which event will be published

```php
public function changeBillingDetails(DistributedBus $eventBus)
{
    $eventBus->publishEvent(
         routingKey: "billing.detailsWereChanged",
         '{"personId":123,"billingDetails":0001}',
         "application/json" 
    );
}
```

### Consuming Distributed Events

General flow behind consuming Distributed Event Messages is to add Distributed attribute on top of our Event Handler. This way we state the intention that this Event Handler should take a part in Event Distribution.

```php
#[Distributed]
#[EventHandler("billing.detailsWereChanged")]
public function registerTicket(BillingDetailsWereChanged $event) : void
{
    // do something with event
}
```

{% hint style="info" %}
To start listening for distributed events under specific key, provide `Distributed` attribute.
{% endhint %}

## Generic Message Distribution

You may distribute generic Message. This allows avoiding registering given Message as Command or Event. This is especially useful, if integration happens not from business requirements (therefore we don't want to include it as Command or Event), but it does happens because of technical requirements.&#x20;

{% hint style="info" %}
Ecotone for example use Distributed Messages for communicating between Applications and Ecotone Pulse to inform whatever given message should be replayed or deleted.
{% endhint %}

### Publishing Distributed Messages

To send Generic Messages we will be using DistributedBus:

```php
interface DistributedBus
{
    public function sendMessage(
        string $targetServiceName, 
        string $targetChannelName, 
        string $payload, 
        string $sourceMediaType = MediaType::TEXT_PLAIN, 
        array $metadata = []
    ): void;
    
(...)
```

* **targetServiceName** - is a [Service Name](../#configuration) of targeted Service
* **targetChannelName** - is a target channel on the destination side

```php
public function changeBillingDetails(DistributedBus $distributedCommandBus)
{
    $distributedCommandBus->sendCommand(
        targetServiceName: "billing",
        targetChannelName": "billing.changeDetails",
        '["personId":"123","billingDetails":"01111"]',
        "application/json"
    );
}
```
