---
description: Message Publisher for publishing to external message brokers
---

# Message Publisher

You're integrating with a third-party service that listens on RabbitMQ — they don't run Ecotone, they have their own wire format. You don't want your business code to know about the AMQP client, the topic name, or the serialization format. **Message Publisher** is a typed interface that publishes raw messages to an external broker; if you swap brokers later, the call sites don't change.

### Modules Providing Support

* [RabbitMQ Module](../../modules/amqp-support-rabbitmq/#message-publisher)

## Message Publisher Interface

```php
interface MessagePublisher
{

    // 1
    public function send(string $data, string $sourceMediaType = MediaType::TEXT_PLAIN) : void;

    // 2
    public function sendWithMetadata(string $data, array $metadata, string $sourceMediaType = MediaType::TEXT_PLAIN) : void;

    // 3
    public function convertAndSend(object|array $data) : void;

    // 4
    public function convertAndSendWithMetadata(object|array $data, array $metadata) : void;
}
```

1. `send` - Send a `string type` via Publisher. It does not need any conversion, you may add additional `Media Type` of `$data`.
2. `sendWithMetadata` - Does the same as `send,` allows for sending additional [Meta data](../../tutorial-php-ddd-cqrs-event-sourcing/php-metadata-method-invocation.md#metadata).
3. `convertAndSend` - Allow for sending types, which needs conversion. Allow for sending objects and array, `Ecotone` make use of [Conversion system](../../messaging/conversion/conversion/) to convert `$data`.
4. `convertAndSendWithMetadata` - Does the same as `convertAndSend,` allow for sending additional [Meta data](../../tutorial-php-ddd-cqrs-event-sourcing/php-metadata-method-invocation.md#metadata).

## How to use Message Publisher

Publisher is a special type of [Gateway](../../messaging/messaging-concepts/messaging-gateway.md), which implements [Publisher interface](../../modules/amqp-support-rabbitmq/#available-actions).\
It will be available in your Dependency Container under passed `Reference name.`\
In case interface name `MessagePublisher:class` is used, it will be available using auto-wire.

```php
#[EventHandler] 
public function whenOrderWasPlaced(OrderWasPlaced $event, MessagePublisher $publisher) : void
{
    $publisher->convertAndSendWithMetadata(
        $event,
        [
            "system.executor" => "Johny"
        ]
    );
}
```
