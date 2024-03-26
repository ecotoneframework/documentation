# Shared and Multi Database Tenants

We may have business model where by default we put every Tenant in the same Database, yet if Customer will buy premium he will receive separate Database instance.

To handle such cases, Ecotone provides the default connection. This way, if there is no mapping for given Tenant name, default will be used:

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function multiTenantConfiguration(): MultiTenantConfiguration
    {
        return MultiTenantConfiguration::createWithDefaultConnection(
            tenantHeaderName: 'tenant',
            tenantToConnectionMapping: [
                'tenant_a' => 'tenant_a_connection',
                'tenant_b' => 'tenant_b_connection'
            ],
            // Provide default connection
            'tenant_default_connection'
        );
    }
}
```

**tenant\_default\_connection** will be used in case, when mapping will be not found for given Tenant.
