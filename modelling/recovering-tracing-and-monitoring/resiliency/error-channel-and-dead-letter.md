# Error Channel and Dead Letter

## Error Channel

`Ecotone` comes with solution that allow to push Error Messages to `error channel`.\
Depending on what subscribing to error channel you may provide different behaviour. \
From logging, storing or even discarding the message.

### Error Channel

The error channel is channel defined for handling failed messages. As a default it's turned off. \
We may set up error channel for specific consumer

* [Set up default error channel for all consumers](../../../messaging/service-application-configuration.md#ecotone-core-configuration)

&#x20;         \- [Symfony](../../../modules/symfony/symfony-ddd-cqrs-event-sourcing.md#defaulterrorchannel)

&#x20;         \- [Laravel](../../../modules/laravel/laravel-ddd-cqrs-event-sourcing.md#defaulterrorchannel)

&#x20;         \- [Lite](../../../modules/ecotone-lite/#withdefaulterrorchannel)

* [Set up for specific consumer](../../asynchronous-handling/#static-configuration)

{% hint style="info" %}
Setting up Error Channel means that [Message Consumer](../../../messaging/contributing-to-ecotone/demo-integration-with-sqs/message-consumer-and-publisher.md#message-consumer) will send Error Message to error channel and then continue handling next messages.\
\
After sending Error Message to error channel, message is considered handled.
{% endhint %}

## Manually Handling Error Messages from Error Channel

After setting it up default error channel to "errorChannel" we may subscribe to the errors by setting up [ServiceActivator](../../../messaging/messaging-concepts/):

```php
#[ServiceActivator("errorChannel")]
public function handle(ErrorMessage $errorMessage): void
{
    // do something with ErrorMessage
}
```

{% hint style="info" %}
Service Activator are endpoints like Command Handlers, however they are not exposed using Command/Event/Query Buses. \
You may use them for internal handling.
{% endhint %}

## Dbal Dead Letter

Ecotone comes with full support for managing full life cycle of a error message by using [Dbal Module](../../../modules/dbal-support.md#dead-letter).

* Store failed Message with all details about the exception
* Allow for reviewing error Messages
* Allow for deleting and replaying error Message back to the [Asynchronous Message Channels](../../asynchronous-handling/)

## Installation

* Install [Ecotone's Dbal Module](../../../modules/dbal-support.md).&#x20;
* Set up Error Channel like discussed at the [beginning of the section](error-channel-and-dead-letter.md#error-channel)
* Set up "_errorChannel"_ in _ErrorHandlerConfiguration to "dbal\_dead\_letter"_

<pre class="language-php"><code class="lang-php">#[ServiceContext]
public function errorConfiguration()
{
    return ErrorHandlerConfiguration::createWithDeadLetterChannel(
        "errorChannel",
        // <a data-footnote-ref href="#user-content-fn-1">your</a> retry strategy
        RetryTemplateBuilder::exponentialBackoff(1000, 10)
            ->maxRetryAttempts(3),
        // if retry strategy will not recover, then send here
        "dbal_dead_letter"
    );
}
</code></pre>

{% hint style="info" %}
If you would set up "nullChannel" in place of "dbal\_dead\_letter", then all Message that can't be retried with success would be discared.
{% endhint %}

## Dead Letter Console Commands

### Help

Get more details about existing commands

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:deadletter:help
```
{% endtab %}

{% tab title="Laravel" %}
```
artisan ecotone:deadletter:help
```
{% endtab %}
{% endtabs %}

### Listing Error Messages

Listing current error messages

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:deadletter:list
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:deadletter:list
```
{% endtab %}

{% tab title="Lite" %}
```php
$list = $messagingSystem->runConsoleCommand("ecotone:deadletter:list", []);
```
{% endtab %}
{% endtabs %}

### Show Details About Error Message

Get more details about given error message

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:deadletter:show {messageId}
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:deadletter:show {messageId}
```
{% endtab %}

{% tab title="Lite" %}
```php
$details = $messagingSystem->runConsoleCommand("ecotone:deadletter:show", ["messageId" => $messageId]);
```
{% endtab %}
{% endtabs %}

### Replay Error Message

Replay error message. It will return to previous channel for consumer to pick it up and handle again.

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:deadletter:replay {messageId}
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:deadletter:replay {messageId}
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand("ecotone:deadletter:replay", ["messageId" => $messageId]);
```
{% endtab %}
{% endtabs %}

### Replay All Messages

Replaying all the error messages.

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:deadletter:replayAll
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:deadletter:replayAll
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand("ecotone:deadletter:replayAll", []);
```
{% endtab %}
{% endtabs %}

### Delete Message

Delete given error message

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:deadletter:delete {messageId}
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:deadletter:delete {messageId}
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand("ecotone:deadletter:delete", ["messageId" => $messageId]);
```
{% endtab %}
{% endtabs %}

### Turn off Dbal Dead Letter

```php
#[ServiceContext]
public function dbalConfiguration()
{
    return DbalConfiguration::createWithDefaults()
        ->withDeadLetter(false);
}
```

## Managing Multiple Ecotone Applications

The above solution requires running Console Line Commands. If we want however, we can manage all our Error Messages from one place using [Ecotone Pulse](../ecotone-pulse-service-dashboard.md).

This is especially useful when we've multiple Applications, so we can go to single place and see if any Application have failed to process Message.

[^1]: 
