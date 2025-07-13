# Final Failure Strategy

Defines how to handle failures when processing messages. This is final failure strategy as it's used in case, when there is no other way to handle the failure. For example, when there is no [retry policy](retries.md), or when the retry policy has reached its maximum number of attempts. Also, when the destination of Error Channel is not defined, or sending to [Error Channel](error-channel-and-dead-letter/) fails.

## Available Strategies

Ecotone provides three final strategies:

* **RESEND** (default) - Message is requeued for another attempt This is default strategy, this should be used to unblock processing. This way next messages can be consumed without system being stuck preventing next business actions from happening.
* **IGNORE** - Message is discarded, processing continues. Non-critical message.
* **STOP** - Consumer stops, message preserved. This strategy can be applied when our system depends heavily on the order of the Messages to work correctly. In that case we can stop the Consumer, resulting in Message being kept in order.

## Configuration

This can be configured on Message Channel level:

```php
AmqpBackedMessageChannelBuilder::create(channelName: 'async')
    ->withFinalFailureStrategy(FinalFailureStrategy::STOP)
```
