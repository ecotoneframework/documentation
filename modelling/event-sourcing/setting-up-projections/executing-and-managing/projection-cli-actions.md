---
description: PHP rebuild and delete projections
---

# Projection CLI Actions

## Projection Actions

### Projection initialization

As projection can be restarted, deleted and created differently. When the projection knows how to setup it itself, it's easy to rebuild it when change is needed.

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:es:initialize-projection {projectionName}
```
{% endtab %}

{% tab title="Laravel" %}
```
artisan ecotone:es:initialize-projectionn {projectionName}
```
{% endtab %}

{% tab title="Lite" %}
```
$messagingSystem->runConsoleCommand("ecotone:es:initialize-projection", ["name" => $projectionName]);
```
{% endtab %}
{% endtabs %}

And inside the projection we need to implement `ProjectionInitialization` to tell `Ecotone` what to do:

```php
#[ProjectionInitialization]
public function initialization() : void
{
$this->connection->executeStatement(<<<SQL
    CREATE TABLE IF NOT EXISTS in_progress_tickets (
        ticket_id VARCHAR(36) PRIMARY KEY,
        ticket_type VARCHAR(25)
    )
SQL);
}
```

{% hint style="info" %}
With Polling projections, your projection will be initialized automatically.\
With Event Driven projections, your projections will not be created on startup, consider running command first.
{% endhint %}

### Resetting/Rebuilding the projection

In order to restart the projection in case we want to provide incompatible change, we can simply reset the projection and it will build up from the beginning.&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:es:reset-projection {projectionName}
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:es:reset-projection {projectionName}
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand("ecotone:es:reset-projection", ["name" => $projectionName]);
```
{% endtab %}
{% endtabs %}

And inside the projection we need to implement `ProjectionReset` to tell `Ecotone` what to do:

```php
#[ProjectionReset]
public function reset() : void
{
$this->connection->executeStatement(<<<SQL
    DELETE FROM in_progress_tickets
SQL);
}
```

### Deleting the projection

If we want to delete the projection&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
bin/console ecotone:es:delete-projection {projectionName}
```
{% endtab %}

{% tab title="Laravel" %}
```php
artisan ecotone:es:delete-projection {projectionName}
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand("ecotone:es:delete-projection", ["name" => $projectionName]);
```
{% endtab %}
{% endtabs %}

And inside the projection we need to implement `ProjectionDelete` to tell `Ecotone` what to do:

```php
#[ProjectionDelete]
public function delete() : void
{
$this->connection->executeStatement(<<<SQL
    DROP TABLE in_progress_tickets
SQL);
}
```

### Manually triggering projection

If we want to manually trigger projection&#x20;

{% tabs %}
{% tab title="Symfony" %}
```
bin/console ecotone:es:trigger-projection {projectionName}
```
{% endtab %}

{% tab title="Laravel" %}
```
artisan ecotone:es:trigger-projection {projectionName}
```
{% endtab %}

{% tab title="Lite" %}
```
$messagingSystem->runConsoleCommand("ecotone:es:trigger-projection", ["name" => $projectionName]);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
This can come in handy is specific situations. E.g. in case of asynchronous event-driven projection failed in the middle of resetting. Instead of resetting it again or waiting for event which will trigger your projection, you can do it manually.
{% endhint %}
