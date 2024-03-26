---
description: Scheduling PHP
---

# Scheduling

`Ecotone` comes with support for running `period tasks` or `cron jobs.`

## Scheduled Method

```php
class NotificationService
{
    #[Scheduled(endpointId: "notificationSender")]
    #[Poller(fixedRateInMilliseconds: 1000)]
    public function sendNotifications(): void
    {
        echo "Sending notifications...\n";
    }
}
```

`endpointId` - it's name which identifies process to run\
`poller` - Configuration how to execute this method [read more in next section](scheduling.md#polling-metadata). \
Above configuration tells `Ecotone` to execute this method every second.

{% tabs %}
{% tab title="Symfony" %}
```php
console ecotone:list
+--------------------+
| Endpoint Names     |
+--------------------+
| notificationSender |
+--------------------+
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:list
+--------------------+
| Endpoint Names     |
+--------------------+
| notificationSender |
+--------------------+
```
{% endtab %}

{% tab title="Lite" %}
```php
$consumers = $messagingSystem->list()
```
{% endtab %}
{% endtabs %}

After setting up Scheduled endpoint we can run the endpoint:

{% tabs %}
{% tab title="Symfony" %}
```php
console ecotone:run notificationSender -vvv
```
{% endtab %}

{% tab title="Laravel" %}
```
artisan ecotone:run notificationSender -vvv
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->run("notificationSender");
```
{% endtab %}
{% endtabs %}

## Scheduled Handler

You can run `Scheduled` for given Handler.\
Right now method return [Message](../../messaging/messaging-concepts/message.md) which is send to given routing.

```php
class CurrencyExchanger
{
    #[Scheduled(requestChannelName: "exchange", endpointId: "currencyExchanger")] 
    #[Poller(fixedRateInMilliseconds=1000)]
    public function callExchange() : array
    {
        return ["currency" => "EUR", "ratio" => 1.23];
    }
}

#[CommandHandler("exchange")] 
public function exchange(ExchangeCommand $command) : void;
```

`requestChannelName` - The channel name to which [Message](../../messaging/messaging-concepts/message.md) should be send.

When the Message will arrive on the Command Handler it will be automatically converted to `ExchangeCommand.` If you want to understand how the conversion works, you may read about it in [Conversion section](../../messaging/conversion/).

## Materials

### Demo implementation

You may find demo implementation [here](https://github.com/ecotoneframework/quickstart-examples/tree/main/Schedule).

### Links

* [Scheduling Execution in PHP](https://blog.ecotone.tech/scheduling-execution-in-php/)
