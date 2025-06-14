# Dbal Dead Letter

## Dbal Dead Letter

Ecotone comes with full support for managing full life cycle of a error message by using [Dbal Module](../../../../modules/dbal-support.md#dead-letter).

* Store failed Message with all details about the exception
* Allow for reviewing error Messages
* Allow for deleting and replaying error Message back to the [Asynchronous Message Channels](../../../asynchronous-handling/)

## Installation

To make use of Dead Letter, we need to have [Ecotone's Dbal Module](../../../../modules/dbal-support.md) installed.

## Storing Messages in Dead Letter

If we configure default error channel to point to **"dbal\_dead\_letter"** then all Error Messages will land there directly

<figure><img src="../../../../.gitbook/assets/Dead Letter (1).png" alt=""><figcaption><p>Storing Error Messages once they failed directly in Database</p></figcaption></figure>

{% tabs %}
{% tab title="Symfony" %}
**config/packages/ecotone.yaml**

```yaml
ecotone:
  defaultErrorChannel: "dbal_dead_letter"
```
{% endtab %}

{% tab title="Laravel" %}
**config/ecotone.php**

```php
return [
    'defaultErrorChannel' => 'dbal_dead_letter',
];
```
{% endtab %}

{% tab title="Lite" %}
```php
$ecotone = EcotoneLite::bootstrap(
    configuration: ServiceConfiguration::createWithDefaults()
        ->withDefaultErrorChannel('dbal_dead_letter')
);
```
{% endtab %}
{% endtabs %}

## Dead Letter with Delayed Retries

We may also want to try to recover before we consider Message to be stored in Dead Letter:

<figure><img src="../../../../.gitbook/assets/dead-letter-with-retry.png" alt=""><figcaption><p>Storing Error Messages in Dead Letter only if retries are exhausted</p></figcaption></figure>

{% tabs %}
{% tab title="Symfony" %}
**config/packages/ecotone.yaml**

```yaml
ecotone:
  defaultErrorChannel: "errorChannel"
```
{% endtab %}

{% tab title="Laravel" %}
**config/ecotone.php**

```php
return [
    'defaultErrorChannel' => 'errorChannel',
];
```
{% endtab %}

{% tab title="Lite" %}
```php
$ecotone = EcotoneLite::bootstrap(
    configuration: ServiceConfiguration::createWithDefaults()
        ->withDefaultErrorChannel('errorChannel')
);
```
{% endtab %}
{% endtabs %}

and then we use inbuilt Retry Strategy:

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

The above solution requires running Console Line Commands. If we want however, we can manage all our Error Messages from one place using [Ecotone Pulse](../../ecotone-pulse-service-dashboard.md).

This is especially useful when we've multiple Applications, so we can go to single place and see if any Application have failed to process Message.

[^1]: 
