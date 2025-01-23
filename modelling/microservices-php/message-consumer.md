# Message Consumer

Message Consumers allows you to connect your Service (application) with other Services, that can be written in different language or not using Ecotone's [Distributed Bus](distributed-bus/).

### Modules Providing Support

* [RabbitMQ Module](../../modules/amqp-support-rabbitmq.md#message-consumer)

## Message Consumer Implementation

Consumer allow use to customize experience to fetch from chosen source:

```php
class Consumer
{
    // 1. Message Consumer
    #[MessageConsumer("consumer")]
    // 2. Method declaration
    public function execute(string $message, array $headers) : void
    {
        // do something with Message
        // if you have converter registered you can type hint exact type you expect
    }
}
```

1. `Message Consumer` - This tells Ecotone to consider given method to be message consumer.\
   First parameter is used as Consumer Name and this is how it will be run.
2. `Method declaration` - Depending how you define your [method declaration](../../messaging/conversion/method-invocation.md), this way you will receive Message. In case of type hint for `string` then Message payload will be delivered to you as string (xml/json however it's formatted).

{% hint style="success" %}
Connecting Message Consumer to external broker, fetching message and converting it given class is done for you.\
\
All you need to do it set up [Extension Object ](message-consumer.md#modules-providing-support)of given type in your [Service Context](../../messaging/service-application-configuration.md) to use Message Consumer with given integration.
{% endhint %}
