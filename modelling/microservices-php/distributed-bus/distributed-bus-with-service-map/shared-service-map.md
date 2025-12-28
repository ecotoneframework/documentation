# Shared Service Map

## **Shared Service Map**

When Service Map is defined as separate shared library. It becomes explicit what Events is given Service interested in. This also makes the process of subscribing to new Event visible for everyone, therefore we avoid hidden coupling that could lead to broken integration.

Depending on how we share the events Queue vs Streaming, we may want to do additional changes to the Service Map

{% hint style="success" %}
When Service Map is defined as separate shared library. It becomes explicit what Events is given Service interested in. This also makes the process of subscribing to new Event visible for everyone, therefore we avoid hidden coupling that could lead to broken integration.
{% endhint %}

### Explicit Mapping

To be as explicit as it's possible, we can use direct mapping for subscription keys

```php
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "order_service_inbound_events",
                        subscriptionKeys: ["ticket_service.management.*"],
              )
}
```

This ensures that we won't receive Events that are not meant to be directed to given Service.

### Non Explicit Mapping

In some cases, especially with existing Services, explicit mapping may not be possible, in this situations, we may need to use additional functionality to add filtering.

#### Queue Channel

In Queue Channel approach, it's external Service that owns that Channel, Publisher is only delivering his own Events there as part of Service Map contract. However, if we would put about "\*" for receiving events, then if Publishing Service would also be the owner of the Channel we would republish his own Messages back to that Channel.

To avoid that we could use **excludePublishingServices:**

```php
public function serviceMap(): DistributedServiceMap
{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "order_service_inbound_events",
                        subscriptionKeys: ["*"],
                        excludePublishingServices: ["ticket_service"]
              )
}
```

#### Streaming Channel

With Streaming Channel, the publishing Service, will actually be the owner of the Channel to which we want to deliver the Message. Therefore for this, if we are about to use the "\*" to publish all the Messages there, reusing the Service Map in other Service would deliver other Events there too. <br>

To avoid that and to be explicit that we want there only Events from particular Service, we can use **includePublishingServices:**

<pre class="language-php"><code class="lang-php"><strong>public function serviceMap(): DistributedServiceMap
</strong>{
    return DistributedServiceMap::initialize()
              ->withEventMapping(
                        channelName: "shared_order_events",
                        subscriptionKeys: ["*"],
                        includePublishingServices: ["order_service"]
              )
}
</code></pre>
