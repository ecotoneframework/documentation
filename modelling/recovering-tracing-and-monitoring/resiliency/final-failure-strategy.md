---
description: Final failure strategy when all message retries are exhausted
---

# Final Failure Strategy

A poison message fails 3 retries and the error channel is also unreachable (DB down, dead-letter table full). What does the consumer do — block? skip? crash? **Final Failure Strategy** is that last decision. Pick wrong and you either lose messages silently or stop processing for everyone.

It is used as the last resort when there is no [retry policy](retries.md) (or retries are exhausted), and when the [Error Channel](error-channel-and-dead-letter/) is not defined or itself fails.

## Available Strategies

Ecotone provides three final strategies:

* **RELEASE** - Message is released back to the Channel for another attempt. This way order will be preserved, yet it can result in processing being blocked if the message keeps failing.
* **RESEND** - Message is resend back to the Channel for another attempt, as a result Message Consumer will be unblock and will be able to continue on next Messages. This way next messages can be consumed without system being stuck.
* **IGNORE** - Message is discarded, processing continues. Can be used for non critical message, to simply ignore failed messages.
* **STOP** - Consumer stops, message is preserved. This strategy can be applied when our system depends heavily on the order of the Messages to work correctly. In that case we can stop the Consumer, resulting in Message still awaiting to be consumed.

## Configuration Message Channel

This can be configured on Message Channel level:

```php
AmqpBackedMessageChannelBuilder::create(channelName: 'async')
    ->withFinalFailureStrategy(FinalFailureStrategy::STOP)
```

{% hint style="success" %}
Default for Message Channels is **resend strategy.**
{% endhint %}

## Configuration Consumer

This can also be configured at the Message Consumer level

* [RabbitMQ Consumer](../../../modules/amqp-support-rabbitmq/rabbit-consumer.md#final-failure-strategy)
* [Kafka Consumer](../../../modules/kafka-support/kafka-consumer.md#final-failure-strategy)

{% hint style="success" %}
Default for Message Consumers is **stop strategy.**
{% endhint %}

