# Symfony Database Connection (DBAL Module)

We can use Ecotone's Symfony integration to reuse Connections that are already defined in your Symfony Application.

## Using existing Connections \[Recommended]

Suppose we already defined connection in our _"doctrine.yaml"_ file:

```yaml
doctrine:
  dbal:
    # If this configuration is not present, then "default" is implicit config
    connections:
      default:
        url: '%env(resolve:DATABASE_DSN)%'
```

Then to use it as our Default Connection, we can use Service Context config:

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function dbalConfiguration()
    {
        return SymfonyConnectionReference::defaultConnection('default');
    }
}
```

{% hint style="success" %}
By reusing the same database connection that our application already uses, we ensure that everything—our database changes and message handling—happens within the same transaction. \
f anything fails, Ecotone automatically rolls back both the data changes and messaging related changes, keeping our system in a consistent state.&#x20;
{% endhint %}

## Using Manager Registry

Configuring Dbal Module with Manager Registry allows to make your Entities work as a [Ecotone's Aggregates](../../modelling/command-handling/state-stored-aggregate/).

Suppose we already defined connection in our _"doctrine.yaml"_ file:

```yaml
doctrine:
  dbal:
    default_connection: tenant_a_connection
    connections:
      some_connection:
        url: '%env(resolve:DATABASE_DSN)%'
        charset: UTF8
  orm:
    auto_generate_proxy_classes: "%kernel.debug%"
    entity_managers:
      some_orm_connection:
        connection: some_connection
        mappings:
          App:
            is_bundle: false
            type: attribute
            dir: '%kernel.project_dir%/src'
            prefix: 'App'
```

Then to use it as our Default Connection, we can use Service Context config:

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function dbalConfiguration()
    {
        return SymfonyConnectionReference::defaultManagerRegistry('some_connection');
    }
}
```

{% hint style="success" %}
If you use Manager Registry Connection, be aware that "doctrine/orm" package need to be installed.
{% endhint %}

## **Using DSN**

If we don't have existing connection defined, we can make use of DSN directly

```yaml
Enqueue\Dbal\DbalConnectionFactory:
   class: Enqueue\Dbal\DbalConnectionFactory
   factory: ["Ecotone\Dbal\DbalConnection", "fromDsn"]
   arguments: ["pgsql://user:password@host:5432/db_name"]
```
