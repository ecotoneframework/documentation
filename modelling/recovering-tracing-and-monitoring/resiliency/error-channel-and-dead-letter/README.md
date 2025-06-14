# Error Channel and Dead Letter

## Error Channel

`Ecotone` comes with solution called Error Channel. \
Error Channel is a place where unrecoverable Errors can go, this way we can preserve Error Messages even if we can't handle anyhow. \
Error Channel may log those Messages, store them in database, push them to some Asynchronous Channel. The what is to be done is flexibile and can be adjusted to Application needs.

## Configuration

Error Channel can be configured per Message Consumer, or globally as default Error Channel for all Message Consumers:

* [Set up default error channel for all consumers](../../../../messaging/service-application-configuration.md#ecotone-core-configuration)

&#x20;         \- [Symfony](../../../../modules/symfony/symfony-ddd-cqrs-event-sourcing.md#defaulterrorchannel)

&#x20;         \- [Laravel](../../../../modules/laravel/laravel-ddd-cqrs-event-sourcing.md#defaulterrorchannel)

&#x20;         \- [Lite](../../../../modules/ecotone-lite/#withdefaulterrorchannel)

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

* [Set up for specific consumer](../../../asynchronous-handling/#static-configuration)

```php
class Configuration
{    
    #[ServiceContext]
    public function configuration() : array
    {
        return [
            // For Message Consumer orders, configure error channel
            PollingMetadata::create("orders")
                 ->setErrorChannelName("errorChannel")
        ];
    }
}
```

{% hint style="info" %}
Setting up Error Channel means that [Message Consumer](../../../../messaging/contributing-to-ecotone/demo-integration-with-sqs/message-consumer-and-publisher.md#message-consumer) will send Error Message to error channel and then continue handling next messages.\
\
After sending Error Message to Error Channel, message is considered handled as long as Error Handler does not throw exception.
{% endhint %}

## Manual Handling

To handle incoming Error Messages, we can bind to our defined Error Channel using [ServiceActivator](../../../../messaging/messaging-concepts/):

```php
#[InternalHandler("errorChannel")]
public function handle(ErrorMessage $errorMessage): void
{
    // handle exception
    $exception = $errorMessage->getException();
}
```

{% hint style="info" %}
Internal Handlers are endpoints like Command Handlers, however they are not exposed using Command/Event/Query Buses. \
You may use them for internal handling.
{% endhint %}

## Delayed Retries

We can also use inbuilt retry mechanism, that will be resend Error Message to it's original Message Channel with delay. If our Default Error Channel is configured for name **"errorChannel"**, then we can connect it like bellow:

```php
#[ServiceContext]
public function errorConfiguration()
{
    return ErrorHandlerConfiguration::create(
        "errorChannel",
        RetryTemplateBuilder::exponentialBackoff(1000, 10)
            ->maxRetryAttempts(3)
    );
}
```

## Discarding all Error Messages

If for some cases we want to discard Error Messages, we can set up error channel to default inbuilt one called **"nullChannel"**. \
That may be used in combination of retries, if after given attempt Message is still not handled, then discard:

```php
#[ServiceContext]
public function errorConfiguration()
{
    return ErrorHandlerConfiguration::createWithDeadLetterChannel(
        "errorChannel",
        RetryTemplateBuilder::exponentialBackoff(1000, 10)
            ->maxRetryAttempts(3),
        // if retry strategy will not recover, then discard
        "nullChannel"
    );
}
```

## Dbal Dead Letter

Ecotone comes with full support for managing full life cycle of a error message. Read more in next [section](dbal-dead-letter.md).
