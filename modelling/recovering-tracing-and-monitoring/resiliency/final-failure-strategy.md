# Final Failure Strategy

Defines how to handle failures when processing messages. This is final failure strategy as it's used in case, when there is no other way to handle the failure. For example, when there is no [retry policy](retries.md), or when the retry policy has reached its maximum number of attempts. Also, when the destination of Error Channel is not defined, or sending to [Error Channel](error-channel-and-dead-letter/) fails.

## Available Strategies

Ecotone provides three final strategies:

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

