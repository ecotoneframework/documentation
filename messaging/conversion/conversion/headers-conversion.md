# Headers Conversion

Ecotone allows for serialization of Message Headers before they go the Message Channel (Queue).

## Scalar Headers

When we are sending Message Headers as simple scalar types like string or integer, then no Conversion is needed. In those situations Message Headers will be passed as they are.&#x20;

```php
// execution
$commandBus->send(
   $command,
   metadata: [
      'executorId' => '1234'
   ]
)
```

This can be then accessed on the Message Handler level:

```php
#[Asynchronous('async')]
#[CommandHandler]
public function placeOrder(
   PlaceOrderCommand $command,
   #[Header('executorId')] string $executorId
)
{
   // do something
}
```

### Converting Scalars on the Message Endpoint

Even so if Header was sent as scalar, we still can convert it to Application specific object on the Message Handler level.&#x20;

```php
#[Asynchronous('async')]
#[CommandHandler]
public function placeOrder(
   PlaceOrderCommand $command,
   #[Header('executorId')] ExecutorId $executorId
)
{
   // do something
}
```

For this we do need a Converter, which will state how given Class should be deserialized from **string** to **ExecutorId** object.

```php
class ExampleConverterService
{
    #[Converter] 
    public function convert(string $data) : ExecutorId
    {
        return ExecutorId::fromString($data);
    }
}
```

## Class Headers

We may actually want to pass Headers in more meaningful representation than simple scalars and let Ecotone do serialization when given Message goes to Message Channel (Queue).

```php
$commandBus->send(
   $command,
   metadata: [
      'executorId' => ExecutorId::fromString('1234')
   ]
)
```

For this we need Conversion from ExecutorId to string, for Ecotone to know how to serialize it before it will go to Asynchronous Channel.

```php
class ExampleConverterService
{
    #[Converter] 
    public function convert(ExecutorId $data) : string
    {
        return $data->toString();
    }
}
```

## Dealing with Collections

There may be cases, when simple PHP to PHP Conversion will not be enough. \
This may be for example, when we will be dealing with Collection of Classes.&#x20;

```php
$commandBus->send(
   $command,
   metadata: [
      'types' => [
         OrderType::fromString('quick-delivery'),
         OrderType::fromString('promotion'),
      ]
   ]
)
```

In those cases Ecotone will fallback to serialize Message Header to JSON. This way we can serialize even most complex Headers.

Then on the receiving side:

```php
/**
* @param OrderType[] $types
*/
#[Asynchronous('async')]
#[CommandHandler]
public function placeOrder(
   PlaceOrderCommand $command,
   #[Header('types')] array $types
)
{
   // do something
}
```

Actually the Docblock is crucial here, as it tell Ecotone what Collection type we are dealing with. This way Ecotone will be able to pass is to your defined [Media Type Converter](./#media-type-converter) to deserialize it.
