# Database Connection (DBAL Module)

We can use Ecotone's simple Connection setup to define Connection to be used by [Dbal Module](../dbal-support.md).

## Using existing Connection

To reuse existing Connection add Service to your existing setup under _DbalConnectionFactory_ name:

```php
$application = EcotoneLite::boostrap(
    containerOrAvailableServices: [
        DbalConnectionFactory::class => DbalConnection::create(
            $connection  // Doctrine\DBAL\Connection
        )
    ]
);
```

## Configuring Connection from DSN

To define the Connection add Service to your existing setup under _DbalConnectionFactory_ name:

```php
$application = EcotoneLite::boostrap(
    containerOrAvailableServices: [
        DbalConnectionFactory::class => DbalConnection::fromDsn('pgsql://user:password@host:5432/db_name')
    ]
);
```
