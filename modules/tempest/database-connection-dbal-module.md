---
description: Database connection with DBAL module in Tempest
---

# Database Connection (DBAL Module)

We can use Ecotone's Tempest integration to reuse the Connection that is already configured in your Application. This is what powers DBAL features such as the transactional outbox, dead letter, deduplication, document store, event sourcing and multi-tenant.

## **Using the existing Tempest Connection \[Recommended]**

In a Tempest application the database is configured through a `database.config.php` file, for example:

```php
use Tempest\Database\Config\PostgresConfig;

return new PostgresConfig(
    host: env('DB_HOST', 'localhost'),
    port: env('DB_PORT', '5432'),
    username: env('DB_USERNAME', 'app'),
    password: env('DB_PASSWORD', ''),
    database: env('DB_DATABASE', 'app'),
);
```

Using a `ServiceContext` we tell Ecotone to reuse this Connection as the default:

```php
use Ecotone\Messaging\Attribute\ServiceContext;
use Ecotone\Tempest\Config\TempestConnectionReference;

final class EcotoneConfiguration
{
    #[ServiceContext]
    public function tempestConnection(): TempestConnectionReference
    {
        return TempestConnectionReference::defaultConnection();
    }
}
```

{% hint style="success" %}
Reusing the same connection as your application uses ensures database transactions will be rolled back correctly in case of any failure — your Tempest model writes and Ecotone's DBAL operations share one connection.\
\
That's all that is needed. Ecotone will now use your Tempest database connection as the default.
{% endhint %}

## Multi-Tenant

For multi-tenant applications, register a tagged `DatabaseConfig` per tenant in your Tempest configuration, and map each tenant to a `TempestConnectionReference` by tag:

```php
use Ecotone\Dbal\MultiTenant\MultiTenantConfiguration;
use Ecotone\Messaging\Attribute\ServiceContext;
use Ecotone\Tempest\Config\TempestConnectionReference;

final class EcotoneConfiguration
{
    #[ServiceContext]
    public function multiTenant(): MultiTenantConfiguration
    {
        return MultiTenantConfiguration::create(
            tenantHeaderName: 'tenant',
            tenantToConnectionMapping: [
                'tenant_a' => TempestConnectionReference::create('tenant_a'),
                'tenant_b' => TempestConnectionReference::create('tenant_b'),
            ],
        );
    }
}
```

A command or query carrying the `tenant` header is then routed to the matching tenant database automatically.

{% hint style="info" %}
The DBAL features require the `ecotone/dbal` package: `composer require ecotone/dbal`.
{% endhint %}
