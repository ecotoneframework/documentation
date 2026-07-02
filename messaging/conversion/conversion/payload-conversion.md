---
description: Converting message payloads between formats in Ecotone PHP
---

# Payload Conversion

To get general idea about conversion refer to previous [section](./). In this we will focus how Ecotone deals with Conversion for Asynchronous processing.&#x20;

## Outbound Messages Conversion

Suppose we have `Command Handler` endpoint, which expects `PlaceOrderCommand` class and Command Handler is considered Asynchronous.

<pre class="language-php"><code class="lang-php"><strong>#[Asynchronous('async')]
</strong><strong>#[CommandHandler("order.place")]
</strong>public function placeOrder(PlaceOrderCommand $command)
{
   // do something
}
</code></pre>

Then after Message is sent using Command Bus, it goes first to the **Message Channel (Queue)**, named **async**. Before Message lands in the Queue, Ecotone will serialize it, and just before method is executed it will deserialize it.\
\


<figure><img src="../../../.gitbook/assets/serialziation.png" alt=""><figcaption><p>Message serialization and deserialization</p></figcaption></figure>

## Default Conversion Media Type

To which format this Command will be converted depends on our **defaultSerializationMediaType**, which can be configured in global configuration:

* [Ecotone Lite](../../../modules/ecotone-lite/#configuration)
* [Symfony](../../../modules/symfony/symfony-ddd-cqrs-event-sourcing.md)
* [Laravel](../../../modules/laravel/laravel-ddd-cqrs-event-sourcing.md)

## Native PHP Serialization

If default conversion will not be set up, the default serialization configuration will be to use PHP Serialization mechanism. \
What is important to understand is that PHP serialization expect class to be exactly the same as it was serialized and Command will land in Message Queue before it will be handled:\


<figure><img src="../../../.gitbook/assets/place_order.png" alt=""><figcaption><p>Place Order lands in Queue, awaiting to be consumed</p></figcaption></figure>

&#x20;The problem with this is, that if we change the Class Name or property name and deploy new version of our Application before this Message will be consumed, we won't be able to deserialize it anymore.\
Therefore PHP Native Serialization mechanism should not be considered as Production grade solution.\
\
There is no such problem when we use decoupled Media Type format like JSON or XML, as those allow for more losely coupled mapping between Serialized Message and PHP Class.\
Ecotone provides [JMS Converter Module](../../../modules/jms-converter.md), which without any additional configuration can do serialization to JSON or XML.

{% hint style="success" %}
Packages like [JMS Converter](../../../modules/jms-converter.md) will change your default serialization configuration to JSON by default. No custom configuration is needed.
{% endhint %}

## Default Content Type for consumed Messages

When Message is consumed, Ecotone reads its **contentType** header to know from which Media Type the payload should be deserialized. However Messages produced by external systems (e.g. published to a Kafka topic by non-Ecotone application) may come without contentType header at all. In such case Ecotone assumes the payload is already in PHP format and skips deserialization, which for typed parameters ends up with lack of converter exception.

For such integrations we can state explicitly what format is expected, using **ContentType** attribute on given endpoint:

```php
#[ContentType('application/json')]
#[KafkaConsumer(
    endpointId: 'orderConsumers', 
    topics: ['orders']
)]
public function handle(Order $payload): void
{
    // payload without contentType header will be deserialized from JSON
}
```

If Message already carries contentType header, it stays untouched — the attribute works as a default. If we want to always enforce given Media Type, we can mark it to replace the existing one:

```php
#[ContentType('application/json', replaceIfExists: true)]
```

{% hint style="info" %}
ContentType attribute is transport agnostic — it can be used on any endpoint: Kafka Consumers, Asynchronous Command and Event Handlers, and others.
{% endhint %}
