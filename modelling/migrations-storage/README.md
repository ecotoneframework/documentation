# Migrations / Storage

Ecotone provides comprehensive migration commands to manage your storage infrastructure, giving you full control over when and how storage resources are created or removed.

## Automatic Initialization (Default)

By default, Ecotone automatically creates required database tables and message channels on demand (first use). This means you can start using Ecotone right away without needing to set up any infrastructure yourself - tables are created when you first store a document, queues are declared when you first send a message, and so on.

This approach works great in most scenarios, especially during development and in applications where initialization state can be cached between requests.

## When to Use Migration Commands

In some scenarios, you may want to take over the initialization process however:

- **Performance-sensitive HTTP applications** - If you send messages to a broker with each HTTP request and cannot cache initialization state between requests, the automatic check may add unnecessary overhead
- **Strict deployment processes** - Some applications follow a process where all migrations must be completed before starting the application
- **Visibility and control** - You want to see what storage resources exist and their initialization state
- **Resource cleanup** - You need to delete storage resources when features are no longer needed
- **Integration with existing tools** - You want to generate SQL statements for use with Doctrine Migrations, Laravel Migrations, or similar tools

For these cases, Ecotone provides migration commands to take over the process.

## Overview

Ecotone offers two types of migration commands:

1. **Database Migrations** - Manage database tables for features like deduplication, dead letter, document store, database message queues, event sourcing, and projections
2. **Channel Migrations** - Manage message channel infrastructure (queues, exchanges, topics) for brokers like RabbitMQ, SQS, Redis, and Kafka

## Disabling Automatic Initialization

Before using migration commands, disable automatic initialization.  
This way we disable automatic creation of tables and channels.

### Database Tables

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function dbalConfiguration(): DbalConfiguration
    {
        return DbalConfiguration::createWithDefaults()
            ->withAutomaticTableInitialization(false);
    }
}
```

### Message Channels

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function channelConfiguration(): ChannelInitializationConfiguration
    {
        return ChannelInitializationConfiguration::createWithDefaults()
            ->withAutomaticChannelInitialization(false);
    }
}
```

## Database Migration Commands

{% hint style="info" %}
Database migration commands require the [DBAL module](../modules/dbal-support.md) to be installed.
{% endhint %}

### Viewing Status

{% tabs %}
{% tab title="Symfony" %}
```bash
# Show all features and their status
bin/console ecotone:migration:database:setup

# Show only used features (default)
bin/console ecotone:migration:database:setup --onlyUsed=true

# Show all available features
bin/console ecotone:migration:database:setup --onlyUsed=false

# Show status for specific features
bin/console ecotone:migration:database:setup --feature=dead_letter --feature=deduplication
```
{% endtab %}

{% tab title="Laravel" %}
```bash
# Show all features and their status
php artisan ecotone:migration:database:setup

# Show only used features (default)
php artisan ecotone:migration:database:setup --onlyUsed=true

# Show all available features
php artisan ecotone:migration:database:setup --onlyUsed=false

# Show status for specific features
php artisan ecotone:migration:database:setup --feature=dead_letter --feature=deduplication
```
{% endtab %}

{% tab title="Lite" %}
```php
// Show all features and their status
$messagingSystem->runConsoleCommand('ecotone:migration:database:setup', []);

// Show only used features (default)
$messagingSystem->runConsoleCommand('ecotone:migration:database:setup', ['onlyUsed' => true]);

// Show all available features
$messagingSystem->runConsoleCommand('ecotone:migration:database:setup', ['onlyUsed' => false]);

// Show status for specific features
$messagingSystem->runConsoleCommand('ecotone:migration:database:setup', [
    'feature' => ['dead_letter', 'deduplication']
]);
```
{% endtab %}
{% endtabs %}

Output displays three columns: Feature name, whether it's used, and whether it's initialized:

```
+----------------+------+-------------+
| Feature        | Used | Initialized |
+----------------+------+-------------+
| dead_letter    | Yes  | No          |
| deduplication  | Yes  | Yes         |
| document_store | No   | No          |
+----------------+------+-------------+
```

### Initializing Tables

{% tabs %}
{% tab title="Symfony" %}
```bash
# Initialize all used features
bin/console ecotone:migration:database:setup --initialize=true

# Initialize specific features only
bin/console ecotone:migration:database:setup --feature=dead_letter --initialize=true

# Initialize all available features (including unused)
bin/console ecotone:migration:database:setup --initialize=true --onlyUsed=false
```
{% endtab %}

{% tab title="Laravel" %}
```bash
# Initialize all used features
php artisan ecotone:migration:database:setup --initialize=true

# Initialize specific features only
php artisan ecotone:migration:database:setup --feature=dead_letter --initialize=true

# Initialize all available features (including unused)
php artisan ecotone:migration:database:setup --initialize=true --onlyUsed=false
```
{% endtab %}

{% tab title="Lite" %}
```php
// Initialize all used features
$messagingSystem->runConsoleCommand('ecotone:migration:database:setup', ['initialize' => true]);

// Initialize specific features only
$messagingSystem->runConsoleCommand('ecotone:migration:database:setup', [
    'feature' => ['dead_letter'],
    'initialize' => true
]);

// Initialize all available features (including unused)
$messagingSystem->runConsoleCommand('ecotone:migration:database:setup', [
    'initialize' => true,
    'onlyUsed' => false
]);
```
{% endtab %}
{% endtabs %}

### Getting SQL Statements

If you want to integrate with existing database migration tools (like Doctrine Migrations or Laravel Migrations), you can get the raw SQL statements:

{% tabs %}
{% tab title="Symfony" %}
```bash
# Get SQL for all used features
bin/console ecotone:migration:database:setup --sql=true

# Get SQL for specific features
bin/console ecotone:migration:database:setup --feature=dead_letter --sql=true
```
{% endtab %}

{% tab title="Laravel" %}
```bash
# Get SQL for all used features
php artisan ecotone:migration:database:setup --sql=true

# Get SQL for specific features
php artisan ecotone:migration:database:setup --feature=dead_letter --sql=true
```
{% endtab %}

{% tab title="Lite" %}
```php
// Get SQL for all used features
$result = $messagingSystem->runConsoleCommand('ecotone:migration:database:setup', ['sql' => true]);

// Get SQL for specific features
$result = $messagingSystem->runConsoleCommand('ecotone:migration:database:setup', [
    'feature' => ['dead_letter'],
    'sql' => true
]);
```
{% endtab %}
{% endtabs %}

### Deleting Tables

{% tabs %}
{% tab title="Symfony" %}
```bash
# Preview what would be deleted (dry run)
bin/console ecotone:migration:database:delete

# Delete all used features
bin/console ecotone:migration:database:delete --force=true

# Delete specific features
bin/console ecotone:migration:database:delete --feature=dead_letter --force=true
```
{% endtab %}

{% tab title="Laravel" %}
```bash
# Preview what would be deleted (dry run)
php artisan ecotone:migration:database:delete

# Delete all used features
php artisan ecotone:migration:database:delete --force=true

# Delete specific features
php artisan ecotone:migration:database:delete --feature=dead_letter --force=true
```
{% endtab %}

{% tab title="Lite" %}
```php
// Preview what would be deleted (dry run)
$messagingSystem->runConsoleCommand('ecotone:migration:database:delete', []);

// Delete all used features
$messagingSystem->runConsoleCommand('ecotone:migration:database:delete', ['force' => true]);

// Delete specific features
$messagingSystem->runConsoleCommand('ecotone:migration:database:delete', [
    'feature' => ['dead_letter'],
    'force' => true
]);
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
The `--force` flag is required to actually delete tables. Without it, the command only shows what would be deleted.
{% endhint %}

### Available Database Features

| Feature | Description |
|---------|-------------|
| `dead_letter` | Failed message storage for error recovery |
| `deduplication` | Message deduplication tracking |
| `document_store` | Document store persistence |
| `message_queue` | DBAL-backed message queue |
| `event_streams` | Event sourcing stream storage |
| `projection_state` | Projection state tracking |
| `projections_v1` | Legacy projection tables |

## Channel Migration Commands

Message channels for brokers like RabbitMQ, SQS, and Kafka can be managed via migration commands.

{% hint style="info" %}
Not all message channels require migration commands:

- **Redis** - Does not require any resources to be created beforehand. Channel creation is fully automatic as it simply inserts values under given keys.
- **DBAL (Database)** - Message channels backed by database are handled through the database migration commands using the `message_queue` feature. There is no need to use channel migration for DBAL channels.

Channel migration commands are primarily useful for **RabbitMQ** (queues, exchanges) and **SQS/Kafka** (queues, topics) where infrastructure must be declared before use.
{% endhint %}

### Viewing Status

{% tabs %}
{% tab title="Symfony" %}
```bash
# Show all channels and their status
bin/console ecotone:migration:channel:setup

# Show status for specific channels
bin/console ecotone:migration:channel:setup --channel=orders --channel=notifications
```
{% endtab %}

{% tab title="Laravel" %}
```bash
# Show all channels and their status
php artisan ecotone:migration:channel:setup

# Show status for specific channels
php artisan ecotone:migration:channel:setup --channel=orders --channel=notifications
```
{% endtab %}

{% tab title="Lite" %}
```php
// Show all channels and their status
$messagingSystem->runConsoleCommand('ecotone:migration:channel:setup', []);

// Show status for specific channels
$messagingSystem->runConsoleCommand('ecotone:migration:channel:setup', [
    'channels' => ['orders', 'notifications']
]);
```
{% endtab %}
{% endtabs %}

Output displays two columns: Channel name and whether it's initialized:

```
+---------------+-------------+
| Channel       | Initialized |
+---------------+-------------+
| orders        | No          |
| notifications | Yes         |
+---------------+-------------+
```

### Initializing Channels

{% tabs %}
{% tab title="Symfony" %}
```bash
# Initialize all channels
bin/console ecotone:migration:channel:setup --initialize=true

# Initialize specific channels
bin/console ecotone:migration:channel:setup --channel=orders --initialize=true
```
{% endtab %}

{% tab title="Laravel" %}
```bash
# Initialize all channels
php artisan ecotone:migration:channel:setup --initialize=true

# Initialize specific channels
php artisan ecotone:migration:channel:setup --channel=orders --initialize=true
```
{% endtab %}

{% tab title="Lite" %}
```php
// Initialize all channels
$messagingSystem->runConsoleCommand('ecotone:migration:channel:setup', ['initialize' => true]);

// Initialize specific channels
$messagingSystem->runConsoleCommand('ecotone:migration:channel:setup', [
    'channels' => ['orders'],
    'initialize' => true
]);
```
{% endtab %}
{% endtabs %}

### Deleting Channels

{% tabs %}
{% tab title="Symfony" %}
```bash
# Preview what would be deleted (dry run)
bin/console ecotone:migration:channel:delete

# Delete all channels
bin/console ecotone:migration:channel:delete --force=true

# Delete specific channels
bin/console ecotone:migration:channel:delete --channel=orders --force=true
```
{% endtab %}

{% tab title="Laravel" %}
```bash
# Preview what would be deleted (dry run)
php artisan ecotone:migration:channel:delete

# Delete all channels
php artisan ecotone:migration:channel:delete --force=true

# Delete specific channels
php artisan ecotone:migration:channel:delete --channel=orders --force=true
```
{% endtab %}

{% tab title="Lite" %}
```php
// Preview what would be deleted (dry run)
$messagingSystem->runConsoleCommand('ecotone:migration:channel:delete', []);

// Delete all channels
$messagingSystem->runConsoleCommand('ecotone:migration:channel:delete', ['force' => true]);

// Delete specific channels
$messagingSystem->runConsoleCommand('ecotone:migration:channel:delete', [
    'channels' => ['orders'],
    'force' => true
]);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Only channels that support managed initialization will appear in migration commands. In-memory channels and channels without explicit infrastructure requirements are not managed by migrations.
{% endhint %}

## Command Reference

### Database Commands

| Command | Options | Description |
|---------|---------|-------------|
| `ecotone:migration:database:setup` | `--features`, `--initialize`, `--sql`, `--onlyUsed` | View status, initialize tables, or get SQL |
| `ecotone:migration:database:delete` | `--features`, `--force`, `--onlyUsed` | Delete database tables |

### Channel Commands

| Command | Options | Description |
|---------|---------|-------------|
| `ecotone:migration:channel:setup` | `--channel`, `--initialize` | View status or initialize channels |
| `ecotone:migration:channel:delete` | `--channel`, `--force` | Delete message channels |

## Testing

For testing purposes, you can use the same migration commands with Ecotone Lite:

```php
use Ecotone\Lite\EcotoneLite;
use Ecotone\Dbal\Configuration\DbalConfiguration;
use Ecotone\Messaging\Channel\Manager\ChannelInitializationConfiguration;

$ecotone = EcotoneLite::bootstrapFlowTesting(
    containerOrAvailableServices: [
        DbalConnectionFactory::class => $connectionFactory,
    ],
    configuration: ServiceConfiguration::createWithDefaults()
        ->withExtensionObjects([
            DbalConfiguration::createWithDefaults()
                ->withAutomaticTableInitialization(false),
            ChannelInitializationConfiguration::createWithDefaults()
                ->withAutomaticChannelInitialization(false),
        ]),
);

// Initialize storage in test setup
$ecotone->runConsoleCommand('ecotone:migration:database:setup', ['initialize' => true]);
$ecotone->runConsoleCommand('ecotone:migration:channel:setup', ['initialize' => true]);

// Run your tests...

// Clean up in teardown
$ecotone->runConsoleCommand('ecotone:migration:database:delete', ['force' => true]);
$ecotone->runConsoleCommand('ecotone:migration:channel:delete', ['force' => true]);
```

