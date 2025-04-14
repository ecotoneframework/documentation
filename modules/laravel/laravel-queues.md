# Laravel Queues

Ecotone comes with Laravel Queues integration. \
This means we can use our Queues as [Message Channels](../../messaging/messaging-concepts/message-channel.md) for asynchronous communication.

### Asynchronous Message Handler

When your [Queue for given Connection is set up,](https://laravel.com/docs/10.x/queues#connections-vs-queues) you may register it in Ecotone as Asynchronous Message Channel.

We register it using Service Context in Ecotone:

```php
final class MessagingConfiguration
{
    #[ServiceContext]
    public function asyncChannel()
    {
        return LaravelQueueMessageChannelBuilder::create(
            "orders",  // Queue name
            "database" // Optional connection name, otherwise default
        );
    }
}
```

After that we can start using it as any other [asynchronous channel](../../modelling/asynchronous-handling/).

```php
#[Asynchronous('orders')]
#[CommandHandler(endpointId:"place_order_endpoint")]
public function placeOrder(PlaceOrder $command): void
{
    // place the order
}
```

### Trigger Command/Event/Query via Ecotone Bus

In order to trigger `Command` `Event` or `Query` Handler, we will be sending the `Message` via given Ecotone's Bus.

{% hint style="info" %}
Ecotone provide Command/Event/Query buses. \
It's important to distinguish between Message types as `Queries` are handled synchronously, `Commands` are point to point and target single Handler, `Events` on other hand are publish-subscribe which means multiple Handlers can subscribe to it.&#x20;
{% endhint %}

In order to trigger given Bus, inject CommandBus, EventBus or QueryBus (they are automatically available after installing Ecotone) and make use of them to send a Message.

```php
final class OrderController
{
    public function __construct(private CommandBus $commandBus) {}

    public function placeOrder(Request $request): Response
    {
        $command = $this->prepareCommand($request);

        $this->commandBus->send($command);

        return view('success');
    }
    
    (...)
```

### Running Message Consumer (Worker)

Instead of using `queue:work`, we will be running Message Consumer using Ecotone's command:

```bash
php artisan ecotone:run orders -vvv
```

{% hint style="success" %}
In case of failures Ecotone's [retry strategy](../../modelling/recovering-tracing-and-monitoring/resiliency/retries.md#delayed-retries) will kick in.
{% endhint %}

## Sending messages via routing

When sending command and events [via routing](broken-reference), it's possible to use non-class types.&#x20;

* Command Handler with command having `array payload`

```php
$this->commandBus->sendWithRouting('order.place', ["products" => [1,2,3]]);
```

```php
#[Asynchronous('orders')]
#[CommandHandler('order.place', 'place_order_endpoint')]
public function placeOrder(array $payload): void
{
    // do something with 
}
```

* Command Handler inside [Aggregate](../../modelling/command-handling/state-stored-aggregate/) with command having `no payload` at all

```php
$this->commandBus->sendWithRouting('order.place', metadata: ["aggregate.id" => 123]);
```

```php
#[Aggregate]
class Order
{
    #[Asynchronous('orders')]
    #[CommandHandler('cancel', 'cancel_order_endpoint')]
    public function cancel(): void
    {
        $this->isCancelled = true;
    }
(...)    
```

## Asynchronous Event Handlers

In case of sending events, we will be using `Event Bus`.\
Ecotone deliver copy of the `Event` to each of the `Event Handlers`, this allows for handling in isolation and safe retries.

* Inject Event Bus into your service, it will be available out of the box.

```php
$this->eventBus->publish(new OrderWasPlaced($orderId));
```

* Subscribe to event

```php
#[Asynchronous('orders')]
#[EventHandler('order_was_placed_endpoint')]
public function notifyAbout(OrderWasPlaced $event): void
{
    // send notification
}
```

## Serializing in different formats

By default all your Messages will be serialized using PHP native serialization. However this is not recommended way, as native PHP serialization requires class to be kept the same on deserialization, if we will change the class name, we will fail to deserialize.&#x20;

\
We may register our our own different [Media Converters](../../messaging/conversion/conversion/), yet you may use inbuilt solution using [Ecotone JMS](../jms-converter.md), to serialize to JSON without any additional configuration.

```php
#[ServiceContext]
public function asyncChannel()
{
    return LaravelQueueMessageChannelBuilder::create("orders")
                ->withDefaultOutboundConversionMediaType(MediaType::createApplicationJson());
}
```

{% hint style="success" %}
If you're using Ecotone JMS, it will automatically set up all your Message Channels to serialize to JSON, as long as you explicitly not state different format.
{% endhint %}

## Dead Letter (Failed Transport)

Failed transport is configured on Ecotone side using Error Channel. Read more in [related section](../../modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/).
