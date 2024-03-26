# Accessing Current Tenant in Message Handler

For specific scenarios we may need to be aware of Tenant’s context in which execution is done. For example given Tenant may have luxury Shop where delivery should happen right away after order is made, where for other Tenant time does not matter.

In case of Ecotone, whatever we send via Message Headers (Metadata) is accessible for us on the Message Handler level. This way depending on the need we can ignore or access given metadata. And as we send Tenant name via Message Headers, we can access it in case of need:

```php
final readonly class OrderService
{
    public function __construct(private FastDelivery $fastDelivery) {}
    
    #[CommandHandler]
    public function handle(
      PlaceOrder $command,
      #[Header('tenant')] $tenantName
    )
    {
        if ($this->fastDelivery->isEnabled($tenantName)) {
            // Place order differently for Premium Tenant    
        }else {
            // Do normal order process
        }
    }
}
```

{% hint style="success" %}
We can access any Message Header in our Message Handlers. This means, whatever Metadata we will pass with Command/Query/Event (e.g. User Id, User Role, HTTP Domain from which request is made etc), we can then access it when needed.
{% endhint %}
