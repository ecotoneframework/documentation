---
description: Running your own Console Commands in the context of a given tenant
---

# Tenant-Aware Console Commands

Console Commands run outside of any incoming HTTP request, so there is no natural place for the tenant to come from. A maintenance script, a nightly report, a backfill - all of them need to state **which tenant they are running for**.

Ecotone solves this with the same mechanism used everywhere else in multi-tenancy: the tenant travels as a **Message Header**. Console Commands accept headers via [`--header={name}:{value}`](../../console-commands.md#passing-message-headers), so a Console Command becomes tenant-aware simply by being given the tenant header on execution - there is no Console Command specific configuration to set up beyond our usual `MultiTenantConfiguration`.

## Example

Our Console Command looks like any other. We inject the Connection for the current Tenant using `#[MultiTenantConnection]`, exactly like we would in a Message Handler:

```php
final class ReportGenerator
{
    #[ConsoleCommand('reports:generate')]
    public function generate(
        #[MultiTenantConnection] Connection $connection
    ): void
    {
        $connection->executeStatement('INSERT INTO reports (generated_at) VALUES (NOW())');
    }
}
```

### Executing Command

We provide the tenant using the tenant header name from our `MultiTenantConfiguration` (here `tenant`):

{% tabs %}
{% tab title="Symfony" %}
```bash
bin/console reports:generate --header="tenant:tenant_a"
```
{% endtab %}

{% tab title="Laravel" %}
```bash
php artisan reports:generate --header="tenant:tenant_a"
```
{% endtab %}

{% tab title="Lite" %}
```php
$messagingSystem->runConsoleCommand('reports:generate', [
    'header' => ['tenant:tenant_a'],
]);
```
{% endtab %}
{% endtabs %}

The injected connection now points at `tenant_a`'s database. Running the same command with `--header="tenant:tenant_b"` writes to `tenant_b`'s database instead - the Console Command code stays identical.

{% hint style="success" %}
This is the same story as the rest of Ecotone's multi-tenancy support: the tenant is context that travels with the Message, and infrastructure resolves the connection from it. Our business code does not branch on tenant, and does not receive a connection manually.
{% endhint %}

## Propagation to Sub-Flows

Message Headers passed to a Console Command are **propagated for the whole execution**. If our Console Command triggers a Command Bus, the tenant context follows automatically:

```php
final class ReportGenerator
{
    #[ConsoleCommand('reports:generate')]
    public function generate(#[Reference] CommandBus $commandBus): void
    {
        $commandBus->send(new GenerateReport());
    }

    #[CommandHandler]
    public function handle(
        GenerateReport $command,
        #[MultiTenantConnection] Connection $connection
    ): void
    {
        // runs against the tenant given on the command line
    }
}
```

Running `bin/console reports:generate --header="tenant:tenant_a"` executes the Command Handler against `tenant_a`'s connection. We do not need to pass the tenant into the Command's payload, or set the metadata on the Bus call by hand.

This holds for deeper flows too - Events published from the handler, asynchronous Message Channels, and any other sub-flow keep the tenant context, exactly as described in [Events and Tenant Propagation](events-and-tenant-propagation.md).

## Missing Tenant Header

If we run a tenant-aware Console Command **without** the tenant header, and no default connection is configured, Ecotone fails loudly:

```
Lack of context about tenant in Message Headers. Please add `tenant` header metadata to your message.
```

This is intentional - a missing tenant surfaces as a clear error instead of silently routing to an arbitrary database. If we do want a fallback for commands that are not tenant specific, we configure it with `MultiTenantConfiguration::createWithDefaultConnection()`, described in [Shared and Multi Database Tenants](shared-and-multi-database-tenants.md).

## Ecotone's Built-in Console Commands

The same `--header` mechanism works for Console Commands shipped by Ecotone itself, which is how we scope them to a single tenant:

```bash
bin/console ecotone:deadletter:list --header="tenant:tenant_a"
bin/console ecotone:deduplication:remove-expired-messages --header="tenant:tenant_a"
```

See [Multi-Tenant aware Dead Letter](multi-tenant-aware-dead-letter.md) for more on tenant-isolated error handling.

{% hint style="info" %}
Running a command for **every** tenant is a matter of looping in our deployment script or scheduler - one invocation per tenant header. Ecotone deliberately does not run a Console Command across all tenants implicitly, so that each execution has one explicit, traceable tenant context.
{% endhint %}
