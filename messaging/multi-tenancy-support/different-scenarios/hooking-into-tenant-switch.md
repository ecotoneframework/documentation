# Hooking into Tenant Switch

If we already have Multi-Tenant application running, most likely we are using some custom libraries or integration. In such cases, it may be required to trigger some code when given Tenant is activated or deactivated.

**Ecotone opens possibility to hook into the process of Tenant switch**, where it can provide Connection that is going to be activated and the Tenant name.

```php
final readonly class HookTenantSwitchSubscription
{
    /** Hooking in tenant activation */
    #[OnTenantActivation]
    public function whenActivated(
        string|ConnectionReference $tenantConnectionName, 
        #[Header('tenant')] $tenantName
    ): void
    {
        echo sprintf("HOOKING into flow: Tenant name %s is about to be activated\n", $tenantName);
    }

    /** Hooking in tenant deactivation */
    #[OnTenantDeactivation]
    public function whenDeactivated(
        string|ConnectionReference $tenantConnectionName, 
        #[Header('tenant')] $tenantName
    ): void
    {
        echo sprintf("HOOKING into flow: Tenant name %s is about to be deactivated\n", $tenantName);
    }
}
```

{% hint style="success" %}
Ecotone follows declarative configuration. This means that we mostly going to state what we want to achieve by marking methods with Attributes. This way we can focus on business part of the system, instead of configuration and setups.
{% endhint %}
