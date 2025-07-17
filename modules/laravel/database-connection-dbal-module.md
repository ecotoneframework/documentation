# Database Connection (DBAL Module)

We can use Ecotone's Laravel integration to reuse Connections that are already defined in your Application.

## **Using Existing Laravel Connection \[Recommended]**

In "**config/database.php"** we do have list of available connections in our Laravel Application, for example:

```php
'connections' => [
    'mysql' => [
        'driver' => 'mysql',
        'url' => env('DATABASE_URL'),
        'host' => env('DB_HOST', '127.0.0.1'),
        'port' => env('DB_PORT', '3306'),
        'database' => env('DB_DATABASE', 'forge'),
        'username' => env('DB_USERNAME', 'forge'),
        'password' => env('DB_PASSWORD', ''),
    ],
(...)
```

with this using ServiceContext we may tell Ecotone to use this Connection:

```php
class EcotoneConfiguration
{
    #[ServiceContext]
    public function laravelConnection(): LaravelConnectionReference
    {
        return LaravelConnectionReference::defaultConnection('mysql');
    }
}
```

{% hint style="success" %}
Reusing same connection as we use in Application, ensures database transactions will be rolled back correctly in case of any failure.\
\
It's all we need to configure. Ecotone will now know to use **"mysql"** as default.
{% endhint %}

## **Using DSN**

If we don't have existing connection defined, we can make use of DSN directly

```php
# Register Service in Provider

use Enqueue\Dbal\DbalConnectionFactory;
use Ecotone\Dbal\DbalConnection;

public function register()
{
     $this->app->singleton(DbalConnectionFactory::class, function () {
         return DbalConnection::fromDsn('pgsql://user:password@host:5432/db_name');
     });
}
```
