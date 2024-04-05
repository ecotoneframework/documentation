---
description: Method Invocation PHP
---

# Method Invocation

## Injecting arguments

`Ecotone` inject arguments to invoked method based on `Parameter Converters`.\
Parameter converters tells `Ecotone` how to resolve specific parameter and what kind of argument is it expecting.&#x20;

Suppose that we have Command Handler [Endpoint](../messaging-concepts/message-endpoint/):

```php
#[CommandHandler] 
public function changePrice(
    #[Payload] ChangeProductPriceCommand $command, 
    #[Headers] array $metadata, 
    #[Reference] UserService $userService
) : void
{
    $userId = $metadata["userId"];
    if (!$userService->isAdmin($userId)) {
        throw new \InvalidArgumentException("You need to be administrator in order to register new product");
    }

    $this->price = $command->getPrice();
}
```

Our Command Handler method declaration is built from three parameters. \
`Ecotone` does resolve parameters based on given attribute types. \
\
`Payload` - Does inject payload of the [message](../messaging-concepts/message.md). In our case it will be the command itself\
`Headers` - Does inject all headers as array.\
`Reference`- Does inject service from Dependency Container. If `referenceName`which is name of the service in the container is not given, then it will take the class name as default.

## Default Converters

`Ecotone`, if parameter converters are not passed provides default converters.&#x20;

* First parameter is always `Payload.`&#x20;
* The second parameter, if is `array` then `Headers` converter is taken
* If class type hint is provided for parameter, then `Reference` converter is picked
* Otherwise, if no default converter can be applied exception will be thrown with information about missing parameter.

Our Command Handler can benefit from default converters, so we don't need to use any additional configuration.

```php
#[CommandHandler] 
public function changePrice(ChangeProductPriceCommand $command, array $metadata, UserService $userService) : void
{
    $userId = $metadata["userId"];
    if (!$userService->isAdmin($userId)) {
        throw new \InvalidArgumentException("You need to be administrator in order to register new product");
    }

    $this->price = $command->getPrice();
}
```

## Parameter Converter Types

### Payload Converter

```php
public function handle(#[Payload] string $payload): void
```

`Payload` converter is responsible for passing payload of the [message](../messaging-concepts/message.md) to given parameter. \
It contains of two attributes:

*   `expression` (Optional) - Allow for performing transformations before passing argument to parameter \`

    ```php
    public function handle(#[Payload("reference('calculatingService').multiply(payload)"] int $amount): void
    ```

{% hint style="success" %}
If don't define attribute, payload will be default converter set up for first method parameter.
{% endhint %}

#### Converting payload

Message's payload is not always the same type as expected in method declaration. \
As in above example, we may expect:&#x20;

```php
ChangeProductPriceCommand $command
```

But the message payload may contains `JSON`:

```php
{"productId": 123, "price": 100}
```

The message may contains of special header `contentType`which describes content type of Message as [media type](https://en.wikipedia.org/wiki/Media\_type). Based on this information, if payload of message is not compatible with parameter's type hint, `Ecotone` do the [conversion](conversion/).

{% hint style="success" %}
Thanks to conversion on the level of endpoint, Ecotone does not expect running `Command Bus` with specific class instance. It may receive anything `xml, json etc` as long as Converter for specific Media Type is registered in the system.
{% endhint %}

#### Expression

Expression does use of great feature of Symfony, called [Expression Language](https://symfony.com/doc/current/components/expression\_language.html).

```php
Payload(expression: "payload * 2")
```

There are three types of variables available within expression.

* `payload` - which is just payload of currently handled Message
* `headers` - contains of all headers available within Message&#x20;
*   `reference` - which allow for retrieving service from Dependency Container and calling a method on it. The result of the expression will be passed to parameter after optional conversion.

    ```php
    Payload(expression:"reference('calculatingService').multiply(payload, 2)")
    ```

### Headers Converter (Headers)

```php
public function handle(#[Headers] array $headers): mixed
```

`Headers` converter is responsible for passing all headers of the [message](../messaging-concepts/message.md) as array to given parameter.&#x20;

{% hint style="success" %}
If don't define attribute, headers will be default converter set up for second method parameter, if is type hinted `array`.
{% endhint %}

### Header Converter (Header)

```php
public function handle(#[Header("executorId")] string $executorId): mixed
```

`Header` converter is responsible for passing specific header from [message](../messaging-concepts/message.md) headers to given parameter. \
It contains following configuration:

* `headerName` (Required) - Allow for performing transformations before passing argument to parameter
* `expression` - Allow for performing transformations before passing argument to parameter, same as in [Payload expression](method-invocation.md#payload-converter-payload)

{% hint style="success" %}
If you type hint Header `nullable`, then header will become optional.\
In case is non-nullable and header does not exists, exception will be thrown.
{% endhint %}

### Reference Converter (DI Service)

```php
public function handle(
    PlaceOrder $command, 
    #[Reference] OrderRepository $orderRepository
): void
```

`Reference` converter is responsible for injecting Service from DI into your method.\
It contains attributes:

* `referenceName` - Allow for defining custom Service Id from DI Containter, if not registered under class name.
* `expression` - Allow for performing transformations before passing argument to parameter, same as in [Payload expression](method-invocation.md#payload-converter-payload)

#### Expression

Expression used with reference can be used for dynamically calling given Service before method execution and injecting an value:

```php
#[Reference(
    referenceName: "globalConfigurationService",
    expression: "service.getCurrentConfiguration()"
)] LocalConfiguration $localConfiguration
```

There are four types of variables available within expression.

* `service` - the service
* `payload` - which is just payload of currently handled Message
* `headers` - contains of all headers available within Message&#x20;
* `reference` - which allow for retrieving service from Dependency Container and calling a method on it. The result of the expression will be passed to parameter after optional conversion.

### Configuration Variable Converter

```php
public function handle(
    #[ConfigurationVariable("isDevelopmentMode")] bool $isDelevopment
): mixed
```

`Configuration Variable` is parameter registered in your configuration.\
It contains following configuration:

* `name` (Optional) - Defines the configuration parameter name, otherwise variable name is taken.

