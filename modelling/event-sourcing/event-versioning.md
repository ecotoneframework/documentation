# Event versioning

In its lifetime events may change. In order to track those changes Ecotone provides possibility of versioning events.&#x20;

```php
use Ecotone\Modelling\Attribute\Revision;

#[Revision(2)]
class MyEvent
{
    public string $id;
}
```

Value given with `Revision` attribute will be stored by Ecotone in events metadata. Attribute is used only when event is saved in event store. In order to read it, you can access events metadata, e.g. in event handler.

```php
use Ecotone\Messaging\MessageHeaders;

class MyEventHandler
{
    #[EventHandler]
    public function handle(MyEvent $event, array $metadata) : void
    {
        if ($metadata[MessageHeaders::REVISION] !== 2) {
            return; // this is not the revision I'm looking for
        }
        
        // the force is strong with this one
    }
}
```

{% hint style="info" %}
`Revision` applies to messages in general (also commands and queries). However, for now it is used only when events gets saved.
{% endhint %}

{% hint style="info" %}
You don't have to define `Revision` for your current events. Ecotone will set it's value to `1` by default. Also, if not defined in the class, already saved events will be read with `Revision` `1`.
{% endhint %}

