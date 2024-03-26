---
description: PHP Converters Serializers Deserializers
---

# JMS Converter

## Installation

{% hint style="success" %}
`composer require ecotone/jms-converter`
{% endhint %}

Ecotone comes with integration with [JMS Serializer](https://jmsyst.com/libs/serializer) and extending it with extra features.

### Module Powered By

Great library, which allow for advanced conversion between types [JMS/Serializer](https://github.com/schmittjoh/serializer).&#x20;

## Native Conversion

Ecotone with JMS will do it's best to deserialize your classes without any additional configuration needed.\
Suppose we have JSON like below:&#x20;

```javascript
{
    "personName": "Johny",
    "address": ["street": "A Good One", "houseNumber": 123
}
```

```php
$this->commandBus->sendWithRouting(
   "settings.change", 
   '{"personName": "Johny","address":["street":"A Good One","houseNumber":123}',
   "application/json"
)

```

Then, suppose we have `endpoint` with following Command:

```php
#[CommandHandler]
public function changeSettings(ChangeSettings $command)
{
   // do something
}
```

```php
class ChangeSettings
{
    private string $personName;    
    private Address $address;
}

class Address
{
    private string $street;
    private string $houseNumber;
}
```

No need for any configuration, deserialization and serialization will be handled for you.

### Array Deserialization

In order to deserialize array you must provide type hint.

```php
/**
* @var Product[]
*/
private array $products;
```

This will let JMS know, how to deserialize given array.\
\
In order to deserialize array of scalar types:

```php
/** @var string[] */
private array $productNames;
=>
["milk","plate"]
```

If your array is hash map however:

```php
/** @var array<string,string> */
private array $order;
=>
["productId":"fff123a","productName":"milk"]
```

If you've mixed array containing scalars, then you may use ArrayObject to deserialize and serialize it preserving keys and types.

```php
private \ArrayObject $data;
=>
["name":"Johny","age":13,"passport":["id":123]]
```

## Custom Conversions To Classes

The difference between Native Conversion is that you take control of deserialization mechanism for specific class. You may call factory method, which will validate correctness of the data or you may provide some default based on your business logic. \
Besides you may find it useful when there is a need to make conversion from `class` to simple type like `string` or `int`.

`JMS Converter`make use of Converters registered as Converters in order to provide all the conversion types described in [Conversion Table](jms-converter.md#conversion-table). You can read how to register new`Converter` in [Conversion section.](../messaging/conversion/conversion.md#conversions-on-php-level)

### Example usage

If we want to call bus with given `JSON` and deserialize `productIds` to `UUID`:

```javascript
{
    "productIds": ["104c69ac-af3d-44d1-b2fa-3ecf6b7a3558"], 
    "promotionCode": "33dab", 
    "quickDelivery": false
}
```

```php
$this->commandBus->sendWithRouting(
   "order.place",  
   '{"productIds": ["104c69ac-af3d-44d1-b2fa-3ecf6b7a3558"], "promotionCode": "33dab", "quickDelivery": false}',
   "application/json"
)
```

Then, suppose we have endpoint with following Command:

```php
#[CommandHandler]
public function placeOrder(PlaceOrder $command)
{
   // do something
}
```

```php
class PlaceOrder
{
    /**
     * @var Uuid[]
     */
    private array $productIds;
    
    private ?string $promotionCode;
    
    private bool $quickDelivery;
}
```

We do not need to add any metadata describing how to convert `JSON to PlaceOrder PHP class.` We already have it using type hints.&#x20;

The only thing, that we need is to add how to convert string to UUID. We do it using Converter:

```php
class ExampleConverterService
{
    #[Converter]
    public function convert(string $data) : Uuid
    {
        return Uuid::fromString($data);
    }
}
```

And that's enough. Whenever we will use `string to UUID` conversion or `string[] to UUID[]`. This converter will be automatically used.&#x20;

## Custom Conversions from Classes

Above example was for deserialization, however if you want to make use of serialization, then Converter from `UUID` to `string` is needed.&#x20;

```php
class ExampleConverterService
{
    #[Converter]
    public function convert(Uuid $data) : string
    {
        return $data->toString();
    }
}
```

```php
class PlaceOrder
{
    /**
     * @var Uuid[]
     */
    private array $productIds;
    
    private ?string $promotionCode;
    
    private bool $quickDelivery;
}

$this->serializer->convertFromPHP(
    new PlaceOrder(//construct), 
    "application/json"
)

=>

{"productIds": ["104c69ac-af3d-44d1-b2fa-3ecf6b7a3558"], "promotionCode": "33dab", "quickDelivery": false}
```

## Serialization Customization

If you want to customize serialization or deserialization process, you may use of annotations on properties, just like it is describes in [Annotation section in JMS Serializer](https://jmsyst.com/libs/serializer/master/reference/annotations).

```php
class GetOrder
{
   /**
   * @SerializedName("order_id")
   */
   private string $orderId;
}
```

## Configuration

Register [Module Conversion](../messaging/service-application-configuration.md#module-configuration)

```php
class Configuration
{
    #[ServiceContext]
    public function getJmsConfiguration()
    {
        return JMSConverterConfiguration::createWithDefaults()
                ->withDefaultNullSerialization(false) // 1
                ->withNamingStrategy("identicalPropertyNamingStrategy"); // 2
    }
}
```

### withDefaultNullSerialization

Should nulls be serialized (`bool, default: false`)

### withNamingStrategy

Serialization naming strategy (`string "identicalPropertyNamingStrategy" || "camelCasePropertyNamingStrategy",` \
`default: "identicalPropertyNamingStrategy"`)&#x20;

## Serialize Nulls for specific conversion

If you want to make convert nulls for [given conversion](../messaging/conversion/conversion.md#serializer), then you can provide Media Type parameters&#x20;

```php
$this->serializer->convertFromPHP(
    ["id" => 1,"name" => null], 
    "application/json;serializeNull=true"
)

=>

{"id":1,"name":null}
```

## Conversion Table

\
`JMS Converter` can handle conversions:

```php
// conversion from JSON to PHP
application/json => application/x-php | {"productId": 1} => new OrderProduct(1)
// conversion from PHP to JSON
application-x-php => application/json | new OrderProduct(1) => {"productId": 1}

// conversion from XML to PHP
application/xml => application/x-php | <productId>1</productId> => new OrderProduct(1)
// conversion from PHP to XML
application-x-php => application/xml | new OrderProduct(1) => <productId>1</productId>

// conversion from JSON to PHP Array
application/json => application/x-php;type=array | {"productId": 1} => ["productId": 1]
// conversion from PHP Array to JSON
application/x-php;type=array => application/json | {"productId": 1} => ["productId": 1]

// conversion from XML to PHP Array
application/xml => application/x-php;type=array | <productId>1</productId> => ["productId": 1]
// conversion from PHP Array to XML
application/x-php;type=array => application/xml | ["productId": 1] => <productId>1</productId>
```
