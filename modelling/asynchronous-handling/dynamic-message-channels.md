# Dynamic Message Channels

\
This chapter provides more details about advanced Message Channel functionalities using Dynamic Message Channels. This may be useful to&#x20;

* Simplify deployment strategy&#x20;
* Optimize system resources&#x20;
* Adjust the configuration per Client, which is especially useful in Multi-Tenant Environments.

{% hint style="success" %}
Dynamic Message Channels are available as part of **Ecotone Enterprise.**
{% endhint %}

## Message Consumption from Multiple Channels

The default strategy is to have single Message Consumer (Worker process) per Message Channel (Queue). When the volume of Messages is low however, some Consumers may actually be continuously in idle state. In order to not be wasteful about system resources we may want then to join the consumption, so single Message Consumer will poll from multiple Message Channels.

Suppose we do have Message Channels:

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function asyncChannels()
    {
        return [
            DbalBackedMessageChannelBuilder::create('orders'),
            DbalBackedMessageChannelBuilder::create('notifications'),
        ];
    }
}
```

To prepare an Message Consumer that will be able to consume from those two Channels in [Round Robin](https://en.wikipedia.org/wiki/Round-robin\_scheduling) manner (meaning each consumption is called after another one) we can set up Dynamic Message Channel.&#x20;

{% hint style="success" %}
Dynamic Message Channels can combine multiple channels, so we can treat them as a one.
{% endhint %}

```php
#[ServiceContext]
public function dynamicChannel()
{
    return [
        DynamicMessageChannelBuilder::createRoundRobin(
            'orders_and_notifications', // channel name to consume
            ['orders', 'notifications']
        ),
    ];
}
```

After that we can consume using **"orders\_and\_notifications"** name. \
We then can run the endpoint:

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:run orders_and_notifications -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:run orders_and_notifications -vvv
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->run("orders_and_notifications");
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
We can combine as many channels as we want under single Dynamic Channel.&#x20;
{% endhint %}

## Distribute based on Context

There may be situations when we would like to introduce Message Channel per Client. This is often an case in Multi-Tenant environments. In Ecotone we can keep our code agnostic of Multiple Channels and keep it focused on the business side, and under the hood implement whatever our Multi-Tenant environment needs. For this we will be using **Header Based Strategy.**

Taking as an example Order Process:

```php
#[Asynchronous("orders")]
#[CommandHandler]
public function placeOrder(PlaceOrderCommand $command) : void
{
   // do something with $command
}
```

This code is fully agnostic to the details of Multi-Tenant environment.  It does use Message Channel "orders" to process the Command. We can however make the "orders" an Dynamic Channel, which will actually distribute to multiple Channels.\
To do this we will introduce we will distribute the Message based on the Metadata from the Command.

```php
#[ServiceContext]
public function dynamicChannel()
{
    return [
        // normal Message Channels
        DbalBackedMessageChannelBuilder::create('tenant_a_channel'),
        DbalBackedMessageChannelBuilder::create('tenant_b_channel'),
    
        // our Dynamic Channel used in Command Handler
        DynamicMessageChannelBuilder::createWithHeaderBasedStrategy(
            'orders',
            'tenant',
            [
                'tenant_a' => 'tenant_a_channel',
                'tenant_b' => 'tenant_b_channel',
            ]
        ),
    ];
}
```

Now whenever this Command is sent with **tenant** metadata, Ecotone will decide to which Message Channel it should proxy the Message.

```php
$this->commandBus->send(
   new PlaceOrderCommand(),
   metadata: [
      'tenant' => 'tenant_a'
   ]
);
```

{% hint style="success" %}
Above will work exactly the same for Events.
{% endhint %}

## Distribute with Default Channel

We may want to introduce separate Message Channels for premium Tenants and just have a shared Message Channel for the rest. For this we would use default channel:

```php
#[ServiceContext]
public function dynamicChannel()
{
    return [
        // normal Message Channels
        DbalBackedMessageChannelBuilder::create('tenant_a_channel'),
        DbalBackedMessageChannelBuilder::create('tenant_b_channel'),
        DbalBackedMessageChannelBuilder::create('shared_channel'),
    
        DynamicMessageChannelBuilder::createWithHeaderBasedStrategy(
            'orders',
            'tenant',
            [
                'tenant_a' => 'tenant_a_channel',
                'tenant_b' => 'tenant_b_channel',
            ],
            'shared_channel' // the default channel, when above mapping does not apply
        ),
    ];
}
```

The default channel will be used, when no mapping will be found:&#x20;

```php
$this->commandBus->send(
   new PlaceOrderCommand(),
   metadata: [
      'tenant' => 'tenant_c' //no mapping for this tenant, go to default channel
   ]
);
```

Then we would run Consumers for all those three channels:

```bash
bin/console ecotone:run tenant_a_channel -vvv
bin/console ecotone:run tenant_b_channel -vvv
bin/console ecotone:run shared_channel -vvv
```

## Speeding up consumption for given Channel

We may want to upgrade above case to provide extra consumption power for given Message Channel. This if often a case in Multi-Tenant environments when premium Customer does get higher consumption rates. For this we can simple run one shared Message Consumer (using Dynamic Channel) like above, plus have extra Message Consumer for specific channels.&#x20;

```php
#[ServiceContext]
public function dynamicChannel()
{
    return [
        DbalBackedMessageChannelBuilder::create('tenant_a_orders'),
        DbalBackedMessageChannelBuilder::create('tenant_b_orders'),
        DbalBackedMessageChannelBuilder::create('tenant_c_orders'),
    
        DynamicMessageChannelBuilder::createRoundRobin(
            'orders',
            ['tenant_a_orders', 'tenant_b_orders', 'tenant_c_orders']
        ),
    ];
}
```

Running shared consumer reading from multiple Channels:

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:run orders -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:run shared_consumer -vvv
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->run("shared_consumer");
```
{% endtab %}
{% endtabs %}

and then running specific consumer for Premium Tenant:

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:run tenant_a_orders -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:run tenant_abc -vvv
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->run("tenant_abc");
```
{% endtab %}
{% endtabs %}

When **shared\_consumer** and **tenant\_abc** will read from same Message Channel at the same time, it will work as [Competitive Consumer pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CompetingConsumers.html). Therefore each will get his own unique Message.&#x20;

## Using Internal Channels

By default whatever Message Channels we will define, we will be able to start Message Consumer for it. However if given set of Channels is only meant to be used under Dynamic Channel, we can make it explicit and avoid allowing them to be run separately.

To do so we use **Internal Channels** which will be available only for the Dynamic Channel visibility.

```php
#[ServiceContext]
public function dynamicChannel()
{
    return [    
            DynamicMessageChannelBuilder::createRoundRobin(
                'orders_and_notifications', // channel name to consume
                ['orders', 'notifications']
            ),
            ->withInternalChannels([
                DbalBackedMessageChannelBuilder::create('orders'),
                DbalBackedMessageChannelBuilder::create('notifications'),
            ]),
    ];
}
```

{% hint style="success" %}
Internal Channels are only visible for the Dynamic Channel, therefore they can't be used for Asynchronous Message Handlers. What should be used for Async Handlers is the name of Dynamic Message Channel.
{% endhint %}

## Using Skipping Strategy

Let's take as an example of Multi-Tenant environment where each of our Clients has set limit of 5 orders to be processed within 24 hours. This limit is known to the Client and he may buy extra processing unit to increase his daily capacity.&#x20;

{% hint style="success" %}
Often used solution to skip processing is to reschedule Messages with a delay and check after some time if Client's message can now be processed. This solution however will waste resources, as we consume Messages that are not meant to be handled. Therefore Ecotone provides alternative, which skips the consumption completely, so we can avoid wasting resources on polling or rescheduling Messages, as we simply don't consume them at all.&#x20;
{% endhint %}

To skip the consumption we will use **Skipping Strategy**. We will start by defining Message Channel per Client, so we can skip consumption from given Channel when Client have reached the limit.&#x20;

```php
#[ServiceContext]
public function dynamicChannel()
{
    return [
        DbalBackedMessageChannelBuilder::create('client_a'),
        DbalBackedMessageChannelBuilder::create('client_b'),
    
        DynamicMessageChannelBuilder::createRoundRobinWithSkippingStrategy(
            'orders',
            'decide_for_client',
            [
                'client_a',
                'client_b',
            ],
        ),
    ];
}
```

The **"decide\_for\_client"** is the name of our [Internal Message Handler](../../messaging/messaging-concepts/message-endpoint/service-activator.md) that will do the decision.

```php
#[InternalHandler('decide_for_client')]
public function decide(
    string $channelNameToCheck, //this will be called in round robin with client_a / client_b
): bool
{
    // We check if client is eligible to consumption    

    // by returning true we start the consumption process, by returning false we skip
    return $this->checkIfRelatedClientHasLimit($channelNameToCheck);
}
```

This function will run in round-robin manner for each defined Message Channel (client\_a, client-b).

{% hint style="success" %}
By using Skipping Strategy we can easily rate limit our Clients. \
This can work dynamically, if Customer will buy credit credits, we can start returning true from decisioning method, which will kick-off the consumption. This means that we create real-time experience for Customers.
{% endhint %}

## Using custom Strategies

In some scenarios we may actually want to take full control over sending and receiving. \
In this situations we can make use of custom Strategies that completely replaces the inbuilt ones. \
This way we can tell which Message Channel we would like to send Message too, and from which Channel we would like to receive Message from.

### Using custom Receiving Strategy

To roll out custom receiving strategy we will use **"withCustomReceivingStrategy":**

```php
#[ServiceContext]
public function dynamicChannel()
{
    return [
        DynamicMessageChannelBuilder::createRoundRobin(
            'orders',
            ['tenant_a', 'tenant_b', 'tenant_c']
        )
            // we change round robin receving strategy to our own customized one
            ->withCustomReceivingStrategy('decide_on_consumption'),
    ];
}
```

To set up our Custom Strategy, we will use [Internal Handler](../../messaging/messaging-concepts/message-endpoint/service-activator.md).&#x20;

```php
final class Decider
{
     /**
     * @return string channel name to consume from
     */
    #[InternalHandler('decide_on_consumption')]
    public function toReceive(): string
    {
        // this should return specific message channel from which we will consume
    }
}
```

{% hint style="success" %}
If we want to stop Consumption completely we can return **"nullChannel"** string. \
This will skip consuming given Channel. \
\
This may be useful in order to turn of given Message Consumer at run time.
{% endhint %}

### Using custom Sending Strategy

To roll out custom receiving strategy we will use **"withCustomReceivingStrategy":**

```php
#[ServiceContext]
public function dynamicChannel()
{
    return [
        DynamicMessageChannelBuilder::createRoundRobin(
            'orders',
            ['tenant_a', 'tenant_b', 'tenant_c']
        )
            // we change round robin sending strategy to our own customized one
            ->withCustomSendingStrategy('decide_on_send'),
    ];
}
```

To set up our Custom Strategy, we will use [Internal Handler](../../messaging/messaging-concepts/message-endpoint/service-activator.md).&#x20;

```php
final class Decider
{
     /**
     * @return string channel name to send too
     */
    #[InternalHandler('decide_on_send')]
    public function toSend(Message $message): string
    {
        // this should return specific message channel to which we send
    }
}
```

{% hint style="success" %}
If we would like to discard given Message, we can return **"nullChannel"** string.
{% endhint %}
