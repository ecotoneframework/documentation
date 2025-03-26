# Resilient Sending

Whenever we use more than one storage during single action, storing to first storage may end up with success, yet the second may not.\
This can happen when we store data in database and then send Messages to Message Broker.\
If failure happen it can be that we will send some Message to Broker, yet fail to store related data or vice versa. \
Ecotone provide you with tools to help solve this problem in order to make sending Messages to Message Broker resilient.

## Message Collector

Ecotone by default enables Message Collector. Collector collect messages that are about to be send to asynchronous channels in order to send them just before the transaction is committed. This way it help avoids bellow pitfalls:

{% hint style="success" %}
Message Collector is enabled by default. It works whenever messages are sent via `Command Bus` or when message are `consumed asynchronously`.
{% endhint %}

### Ghost Messages

Let's consider example scenario: During order processing, we publish an _OrderWasPlaced_ event, yet we fail to store Order in the database. This means we've published Message that is based on not existing data, which of course will create inconsistency in our system.

When Message Collector is enabled it provides much higher assurance that Messages will be send to Message Broker only when your flow have been successful.&#x20;

### Eager Consumption

Let's consider example scenario: During order processing, we may publish an _OrderWasPlaced_ event, yet it when we publish it right away, this Message could be consumed and handled before Order is actually committed to the database. In such situations consumer will fail due to lack of data or may produce incorrect results.&#x20;

Due to Message Collector we gracefully reduce chance of this happening.&#x20;

### Failure on Sending next Message

In general sending Messages to external broker is composed of three stages:

* Serialize Message Payload
* Map and prepare Message Headers
* Send Message to external Broker

In most of the frameworks those three steps are done together, which may create an issue.\
Let's consider example scenario: We send multiple Messages, the first one may with success and the second fail on serialization. Due to that transaction will be rolled back, yet we already produced the first Message, which becomes an Ghost Message.\
&#x20;\
To avoid that Ecotone perform first two actions first, then collect all Messages and as a final step iterate over collected Messages and sent them. This way Ecotone ensures that all Messages must have valid serialization before we actually try to send any of them.

### Disable Message Collector:

As Collector keeps the Messages in memory till the moment they are sent, in case of sending a lot of messages you may consider turning off Message Collector, to avoid memory consumption. \
This way Messages will be sent instantly to your Message Broker.

```php
#[ServiceContext]
public function asyncChannelConfiguration()
{
    return GlobalPollableChannelConfiguration::createWithDefaults()
        ->withCollector(false);
}
```

## Sending Retries

Whenever sending to Message Broker fails, Ecotone will retry in order to self-heal the application.

{% hint style="success" %}
By default Ecotone will do `2` reties when sending to Message Channel fails: \
\- First after `10`ms \
\- Second after `400`ms.
{% endhint %}

You may configure sending retries per asynchronous channel:

```php
#[ServiceContext]
public function asyncChannelConfiguration()
{
    return GlobalPollableChannelConfiguration::create(
        RetryTemplateBuilder::exponentialBackoff(initialDelay: 10, multiplier: 2)
            ->maxRetryAttempts(3)
            ->build()
    );
}
```

## Unrecoverable Sending failures

After exhausting limit of retries in order to send the Message to the Broker, we know that we won't be able to do this. In this scenario instead of letting our action fail completely, we may decide to push it to Error Channel instead of original targetted channel.

```php
#[ServiceContext]
public function asyncChannelConfiguration()
{
    return GlobalPollableChannelConfiguration::createWithDefaults()
            ->withErrorChannel("dbal_dead_letter")
}
```

### Custom handling

```php
#[ServiceContext]
public function asyncChannelConfiguration()
{
    return GlobalPollableChannelConfiguration::createWithDefaults()
            ->withErrorChannel("failure_channel")
}

---

#[ServiceActivator('failure_channel')]
public function doSomething(ErrorMessage $errorMessage): void
{
    // Handle failure message on your own terms :)
}
```

### Dbal Dead Letter

We may decide for example to push it to Dead Letter to store it and later retry:

```php
#[ServiceContext]
public function asyncChannelConfiguration()
{
    return GlobalPollableChannelConfiguration::createWithDefaults()
            ->withErrorChannel("dbal_dead_letter")
}
```

{% hint style="success" %}
If you will push Error Messages to [Dbal Dead Letter](error-channel-and-dead-letter/#dbal-dead-letter), then they will be stored in your database for later review. You may then delete or replay them after fixing the problem. This way we ensure consistency even if unrecoverable failure happened our system continues to have self-healed.
{% endhint %}

## Customized configuration per Message Consumer type

If you need customization per Message Consumer you may do it using `PollableChannelConfiguration` by providing Message Consumer name:

```php
#[ServiceContext]
public function asyncChannelConfiguration()
{
    return PollableChannelConfiguration::createWithDefaults('notifications')
            ->withCollector(false)
            ->withErrorChannel("dbal_dead_letter")
}
```

## Ensure full consistency

For mission critical scenarios, you may consider using [Ecotone's Outbox Pattern](outbox-pattern.md).
