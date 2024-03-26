# How Ecotone works under the hood

## Ecotone's Heart (Configuration)

The heart of `Ecotone` is [Messaging Configuration](https://github.com/ecotoneframework/ecotone-dev/blob/main/packages/Ecotone/src/Messaging/Config/Configuration.php):

```php
Ecotone\Messaging\Config\Configuration::class
```

This is a place, where all messaging concepts are registered. \
Messaging System Configuration is responsible for exposing an API for registering lower level messaging concepts, like [Message Channels](../messaging-concepts/message-channel.md) or [Message Handlers](../messaging-concepts/message-endpoint/).\
\
This is the only place, where all configuration is gathered together in order to prepare [Configured Messaging System](how-ecotone-works-under-the-hood.md#ecotones-heart-in-work-configured-messaging-system).\
\
After setting up configuration, this config can then be cached and reused between requests.\
This is one of the powers of Ecotone, as once the configuration is set up, then all we do is to execute it.

## Ecotone's Heart is beating (Configured Messaging System)

When configuration is prepared, we can then build Configured Messaging System from it:&#x20;

```php
Ecotone\Messaging\Config\Configuration::buildMessagingSystemFromConfiguration()
```

The result of the built is&#x20;

```php
Ecotone\Messaging\Config\ConfiguredMessagingSystem::class
```

This is actually an API that you work with on daily basics when using Ecotone.\
We can get `Command/Query/Event buses` (which are actually [messaging gateways](../messaging-concepts/messaging-gateway.md)) from here, execute `consumer` or fetch `message channel`.

## What the Heart is actually built from?

So we know that the heart of Ecotone is `Messaging Configuration`, what is this built from actually? Who registers Gateways, Message Handlers, Channels?\
And the answer are `Modules`.\
\
`Modules` are layer which transforms User land code to Ecotone's configuration.

{% hint style="info" %}
`Modules` are grouped under `package`. This way you can steer turning off and on set of related modules. \
You may take a look on `Ecotone\Messaging\Config\ModulePackageList` to check all available module packages. \
And `Ecotone\Messaging\Config\ModuleClassList for all modules in given package.`
{% endhint %}

In most of the cases they look for code that is annotated with given Attribute, to register [Messaging Concepts](../messaging-concepts/). This way Ecotone joins messaging world with user land world and provides easy to work with API on the high level, using powerful messaging concepts under the hood.

## Ecotone's Veins

So what happens when we execute gateway like Command Bus?\
It goes through `Message Channels` (`Ecotone\Messaging\MessageChannel`), where at the end of each channel there is specific `Message Handler` (`Ecotone\Messaging\MessageHandler`) implementation. \
You can imagine `channels` as `veins`, that pass `messages` to your `organs` (`Message Handlers`).&#x20;

{% hint style="success" %}
Message Handlers can have input and output message channels, however they unaware of specific Channel implementation. This allow you to switch different implementations like `RabbitMQ,Dbal,In Memory`and your Handlers will stay the same.
{% endhint %}

There are multiple implementations of Message Handlers, including [Service Activator](../messaging-concepts/message-endpoint/service-activator.md), Message Splitter, [Router](../messaging-concepts/message-endpoint/message-routing.md), you will find them in `Ecotone\Messaging\Handler` namespace.

{% hint style="info" %}
What about `Command/Event/Query` Handlers?\
This is just another level of abstraction built on top of messaging concepts. \
In reality `Command Handler` is just an `Service Activator`.\
\
The same about `Command/Event/Query` buses, on the fundamental level they are [Gateways](../messaging-concepts/messaging-gateway.md) combined with[`Routers`](../messaging-concepts/message-endpoint/message-routing.md).
{% endhint %}

So each of the Handlers after receiving message performs an action and may provide reply message. If given Handler has output channel defined, it will go to the next channel.\
This happens till the moment there is no output channel defined and then flows stop.&#x20;

{% hint style="info" %}
Actually, if there is no output channel defined for Message Handler, Message Headers are checked for `replyChannel` header inside the Message. If it exists, it will be used as output channel.\
This is case for receiving reply message when using `Query Bus`.\
\
This logic exists in same place for all Handlers, which is `Ecotone\Messaging\Handler\RequestReplyProducer`.
{% endhint %}
