# Multi-Tenant aware Dead Letter

When executing [Dead Letter Console Command](../../../modelling/recovering-tracing-and-monitoring/resiliency/error-channel-and-dead-letter/#dead-letter-console-commands) we will lack of Context of the Tenant we are using.\
To handle that we may use Ecotone's [support for passing Message Headers](../../console-commands.md#passing-message-headers) together with Console Command.

### Example

Listing current error messages for non-tenant applications would look like this:

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:deadletter:list
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:deadletter:list
```
{% endtab %}

{% tab title="Lite" %}
```php
$list = $messagingSystem->runConsoleCommand("ecotone:deadletter:list", []);
```
{% endtab %}
{% endtabs %}

However if we want to provide context of the Tenant, then we can execute it in following way:

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:deadletter:list --header="tenant:tenant_a"
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:deadletter:list --header="tenant:tenant_a"
```
{% endtab %}

{% tab title="Lite" %}
```php
$list = $messagingSystem->runConsoleCommand("ecotone:deadletter:list", [
    'tenant' => ["tenant:tenant_a"]
]);
```
{% endtab %}
{% endtabs %}

