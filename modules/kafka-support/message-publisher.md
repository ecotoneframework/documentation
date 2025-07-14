# Message Publisher

## Custom Publisher

To create [custom publisher or consumer](../../modelling/microservices-php/) provide [Service Context](../../messaging/service-application-configuration.md).

{% hint style="success" %}
Custom Publishers and Consumers are great for building integrations for existing infrastructure or setting up a customized way to communicate between applications. With this you can take over the control of what is published and how it's consumed.
{% endhint %}

### Custom Publisher

```php
class MessagingConfiguration
{
    #[ServiceContext] 
    public function distributedPublisher()
    {
        return KafkaPublisherConfiguration::createWithDefaults(
            topicName: 'orders'
        );
    }
}
```

Then Publisher will be available for us in Dependency Container under **MessagePublisher** reference.\
This will make it available in your dependency container under **MessagePublisher** name.

### Providing custom rdkafka configuration

You can modify your Message Publisher with specific [rdkafka configuration](https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md):

```php
#[ServiceContext] 
public function distributedPublisher()
{
    return KafkaPublisherConfiguration::createWithDefaults(
        topicName: 'orders'
    )
        ->setConfiguration('request.required.acks', '1');
}
```
