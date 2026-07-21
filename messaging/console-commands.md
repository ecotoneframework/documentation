---
description: Console commands for managing Ecotone message consumers and projections
---

# Console Commands

Ecotone provides support for creating Console Commands. \
Just like the other parts of Ecotone's modules, we register Command in decoupled way using Attributes. This way creating new Console Commands become effortless and clean from extending or implementing framework specific classes and can be placed in code wherever it feels best in given context.

## Commands with Arguments

We register new Console Command using _ConsoleCommand_ attribute:

```php
class EmailSender
{
    #[ConsoleCommand('sendEmail')]
    public function execute(string $email, string $type): void
    {
    }
}
```

Ecotone will register given under _"sendEmail"_ name with two arguments _"email"_ and _"type"_.

### Executing Command

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console sendEmail "test@example.com" "welcome"
```
{% endtab %}

{% tab title="Laravel" %}
```bash
php artisan sendEmail "test@example.com" "welcome"
```
{% endtab %}

{% tab title="Lite" %}
```bash
$messagingSystem->runConsoleCommand(
   'sendEmail',
   [
      "email" => "test@example.com",
       "type" => "welcome"
   ]
)
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Ecotone will register method parameters as Command arguments.
{% endhint %}

## Command Description

We may provide description for our Console Command, which will be visible in the Command listing of given framework (`bin/console list`, `php artisan list`):

```php
class EmailSender
{
    #[ConsoleCommand('sendEmail', 'Sends email of given type to provided address')]
    public function execute(string $email, string $type): void
    {
    }
}
```

## Commands with Options

We register new Console Command using _ConsoleCommand_ attribute:

```php
class EmailSender
{
    #[ConsoleCommand('sendEmail')]
    public function execute(string $email, #[ConsoleParameterOption] string $type): void
    {
    }
}
```

Ecotone will register given under _"sendEmail"_ name with two arguments _"email"_ and _"type"_.

### Executing Command

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console sendEmail "test@example.com" --type="welcome"
```
{% endtab %}

{% tab title="Laravel" %}
```bash
php artisan sendEmail "test@example.com" --type="welcome"
```
{% endtab %}

{% tab title="Lite" %}
```bash
$messagingSystem->runConsoleCommand(
   'sendEmail',
   [
      "email" => "test@example.com",
       "type" => "welcome"
   ]
)
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Ecotone will register any method paramter with attribute _ConsoleParameterOption_ as parameter and all boolean and array type hinted paramters too.
{% endhint %}

## Providing default values

You may provide default value for your parameters, so there will be no need to pass them if not needed:

We register new Console Command using _ConsoleCommand_ attribute:

```php
class EmailSender
{
    #[ConsoleCommand('sendEmail')]
    public function execute(string $email, string $type = 'normal'): void
    {
    }
}
```

### Executing Command

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console sendEmail "test@example.com"
```
{% endtab %}

{% tab title="Laravel" %}
```bash
php artisan sendEmail "test@example.com"
```
{% endtab %}

{% tab title="Lite" %}
```bash
$messagingSystem->runConsoleCommand(
   'sendEmail',
   [
      "email" => "test@example.com"
   ]
)
```
{% endtab %}
{% endtabs %}

## Array of Options

When needed we can expect array of Options to be passed:

```php
class EmailSender
{
    #[ConsoleCommand('sendEmail')]
    public function execute(string $email, array $type): void
    {
    }
}
```

### Executing Command

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console sendEmail "test@example.com" --type=normal --type=test
```
{% endtab %}

{% tab title="Laravel" %}
```bash
php artisan sendEmail "test@example.com" --type=normal --type=test
```
{% endtab %}

{% tab title="Lite" %}
```bash
$messagingSystem->runConsoleCommand(
   'sendEmail',
   [
      "email" => "test@example.com",
      "type" => ["normal", "test"],
   ]
)
```
{% endtab %}
{% endtabs %}

## Passing Services

When given Service is only needed for execution of specific Console Command, there is no need to pass it via constructor. We can scope the injection and inject it directly to our Console Command:

```php
class EmailSender
{
    #[ConsoleCommand('sendEmail')]
    public function execute(
        string $email, 
        string $type, 
        #[Reference] EmailSender $emailSender
    ): void
    {
    }
}
```

Using Reference attribute, we can inject any Service available in Dependency Container, to our Console Command method.

## Passing Message Headers

When running Console Commands we may pass additional Message Headers. This way we can provide context, which may be needed in order to handle given Console Command or for later sub-flows (Message Headers are automatically propagated).

```php
class EmailSender
{
    #[ConsoleCommand('sendEmail')]
    public function execute(
        string $email, 
        string $type, 
        #[Header('token')] string $token
    ): void
    {
    }
}
```

### Executing Command

We pass Message Headers in follow format `--header={name}:{value}`. We may passs as many Message Headers as we want.

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console sendEmail "test@example.com" --type=normal --header="token:123"
```
{% endtab %}

{% tab title="Laravel" %}
```bash
php artisan sendEmail "test@example.com" --type=normal --header="token:123"
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand(
   'sendEmail',
   [
      "email" => "test@example.com",
      "type" => "normal",
      "header" => ["token:123"],
   ]
)
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Headers passed this way are propagated for the whole execution, so they also reach infrastructure that resolves context from Message Headers - for example the [Multi-Tenant connection](multi-tenancy-support/), which picks the active tenant from the configured tenant header. This makes it possible to run tenant-aware Console Commands, e.g. `--header="tenant:tenant_a"`. See [Tenant-Aware Console Commands](multi-tenancy-support/different-scenarios/tenant-aware-console-commands.md) for a full example.
{% endhint %}

## Writing to the Console

To write output from our Console Command, we type hint for _ConsoleWriter_. Ecotone will inject writer connected to the console of given framework - Symfony Console, Laravel Artisan or Tempest Console. This way our Command stays decoupled from framework specific classes, yet output lands in the actual console with native formatting:

```php
use Ecotone\Messaging\Console\ConsoleWriter;

class EmailSender
{
    #[ConsoleCommand('sendEmail')]
    public function execute(string $email, ConsoleWriter $writer): void
    {
        $writer->info("Sending email to {$email}");
        
        $writer->success('Email was sent');
        $writer->warning('Email provider is close to rate limit');
        $writer->error('Email could not be sent');
    }
}
```

_info_, _success_, _warning_ and _error_ write colored messages (cyan, green, yellow, red), while _write_ and _writeln_ write plain text.

{% hint style="success" %}
In Ecotone Lite output is written directly to the standard output as plain text.
{% endhint %}

### Tables

We may render table using column headers and rows:

```php
#[ConsoleCommand('listEmails')]
public function execute(ConsoleWriter $writer): void
{
    $writer->table(
        ['Email', 'Status'],
        [
            ['test@example.com', 'sent'],
            ['other@example.com', 'pending'],
        ]
    );
}
```

### Progress Bars

For long running Commands we may show the progress:

```php
#[ConsoleCommand('sendNewsletter')]
public function execute(ConsoleWriter $writer): void
{
    $subscribers = $this->subscribers();

    $progressBar = $writer->progressBar(count($subscribers));
    foreach ($subscribers as $subscriber) {
        $this->send($subscriber);
        $progressBar->advance();
    }
    $progressBar->finish();
}
```

### Testing Console Output

During flow testing output is collected in memory, so we can simply assert against it:

```php
$ecotoneLite = EcotoneLite::bootstrapFlowTesting([EmailSender::class], [new EmailSender()]);

$ecotoneLite->runConsoleCommand('sendEmail', ['email' => 'test@example.com']);

$this->assertSame(
    ['Email was sent'],
    $ecotoneLite->getInMemoryConsoleWriter()->getSuccessLines()
);
```

## Database Transaction

When Dbal Module is enabled it will automatically wrap your Command in Database Transaction. \
When you want to trigger CommandBus which is wrapped in it's own transaction, it may have sense to turn transactions for Console Commands off:

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function dbalConfiguration()
    {
        return [
            DbalConfiguration::createWithDefaults()
                ->withTransactionOnConsoleCommands(false);
        ];
    }
}
```
