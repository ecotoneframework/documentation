---
description: PHP Event Sourcing Projection Lifecycle and CLI
---

# Lifecycle Management

## The Problem

Your projection's table schema changed, or you found a bug in a handler and the read model has wrong data. How do you set up, tear down, and rebuild projections without writing manual SQL scripts?

Ecotone provides lifecycle hooks — methods on your projection class that are called at specific moments — and CLI commands to trigger them.

## Lifecycle Hooks

### Initialization

Called when the projection is first set up. Use it to create tables, indexes, or any storage structure:

```php
#[ProjectionInitialization]
public function init(): void
{
    $this->connection->executeStatement(<<<SQL
        CREATE TABLE IF NOT EXISTS ticket_list (
            ticket_id VARCHAR(36) PRIMARY KEY,
            ticket_type VARCHAR(25),
            status VARCHAR(25)
        )
    SQL);
}
```

{% hint style="info" %}
By default, projections auto-initialize on the first event trigger. You don't need to run initialization manually unless you use `#[ProjectionDeployment(manualKickOff: true)]`.
{% endhint %}

### Delete

Called when the projection is permanently removed. Clean up all storage:

```php
#[ProjectionDelete]
public function delete(): void
{
    $this->connection->executeStatement('DROP TABLE IF EXISTS ticket_list');
}
```

### Reset

Called when the projection needs to be rebuilt from scratch. Clear the data but keep the structure:

```php
#[ProjectionReset]
public function reset(): void
{
    $this->connection->executeStatement('DELETE FROM ticket_list');
}
```

After a reset, the projection's position is set back to the beginning. The next trigger will replay all events from the start.

### Flush

Called after each batch of events is processed. Useful for flushing buffers or intermediate state:

```php
#[ProjectionFlush]
public function flush(): void
{
    // Called after each batch commit
    // Useful for clearing caches, flushing buffers, etc.
}
```

See [Execution Modes](execution-modes.md#batch-size-and-flushing) for how batching and flushing work together.

## CLI Commands

### Initialize a Projection

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:projection:init ticket_list
# Or initialize all projections at once:
bin/console ecotone:projection:init --all
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:init ticket_list
# Or initialize all:
artisan ecotone:projection:init --all
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand('ecotone:projection:init', ['name' => 'ticket_list']);
```
{% endtab %}
{% endtabs %}

### Delete a Projection

Calls `#[ProjectionDelete]` and removes all tracking state:

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:projection:delete ticket_list
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:delete ticket_list
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand('ecotone:projection:delete', ['name' => 'ticket_list']);
```
{% endtab %}
{% endtabs %}

### Backfill a Projection

Populates a fresh projection with historical events. See [Backfill and Rebuild](backfill-and-rebuild.md) for details:

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:projection:backfill ticket_list
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:backfill ticket_list
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand('ecotone:projection:backfill', ['name' => 'ticket_list']);
```
{% endtab %}
{% endtabs %}

### Rebuild a Projection (Enterprise)

Resets the projection and replays all events. See [Backfill and Rebuild](backfill-and-rebuild.md) for details:

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console ecotone:projection:rebuild ticket_list
```
{% endtab %}

{% tab title="Laravel" %}
```bash
artisan ecotone:projection:rebuild ticket_list
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand('ecotone:projection:rebuild', ['name' => 'ticket_list']);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
The rebuild command is available as part of Ecotone Enterprise.
{% endhint %}

## Automatic vs Manual Initialization

By default, projections auto-initialize the first time an event triggers them. This means you don't need to run any CLI command — the `#[ProjectionInitialization]` method is called automatically.

If you need manual control (for example, during [blue-green deployments](blue-green-deployments.md)), you can disable auto-initialization:

```php
#[ProjectionV2('ticket_list')]
#[FromAggregateStream(Ticket::class)]
#[ProjectionDeployment(manualKickOff: true)]
class TicketListProjection
{
    // Won't auto-initialize — requires explicit CLI init
}
```

{% hint style="info" %}
`#[ProjectionDeployment]` is available as part of Ecotone Enterprise.
{% endhint %}

## Reset and Trigger

To rebuild a projection manually, you can reset it (clears data and position) and then trigger it (starts processing from the beginning):

1. **Reset** — calls `#[ProjectionReset]`, clears position to the beginning
2. **Trigger** — starts processing events from position 0, rebuilding the entire Read Model

This is useful when you've fixed a bug in a handler and need to reprocess all events to correct the data.
