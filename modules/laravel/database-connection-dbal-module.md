# Database Connection (DBAL Module)

We can use Ecotone's Laravel integration to reuse Connections that are already defined in your Application.

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
