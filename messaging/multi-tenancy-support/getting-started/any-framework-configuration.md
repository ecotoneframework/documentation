# Any Framework Configuration

## Configuring Connections

Configuring Multi-Tenancy is straight forward, all we need to do is to define Conncetions for Tenants and mapping.

### For new Connections

For new connections, you may register in your DI Container DbalConnection using DSN.

```php
"tenant_a_connection": Ecotone\Dbal\DbalConnection::fromDsn(getenv("TENANT_A_DATABASE_URL"));
"tenant_b_connection": Ecotone\Dbal\DbalConnection::fromDsn(getenv("TENANT_B_DATABASE_URL"));
```

and then we can setup Mapping

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function multiTenantConfiguration(): MultiTenantConfiguration
    {
        return MultiTenantConfiguration::create(
            tenantHeaderName: 'tenant',
            tenantToConnectionMapping: [
                'tenant_a' => 'tenant_a_connection',
                'tenant_b' => 'tenant_b_connection'
            ],
        );
    }
} 
```

## Sending Message in Context of Tenant

\`We've defined **tenantHeaderName** as **tenant** in our Mapping configuration. This means we can now pass tenant context under this name using Message Headers (metadata).

```php
final readonly class CustomerController extends Controller
{
    public function __construct(private CommandBus $commandBus) {}
    
    public function placeOrder(Request $request): Response
    {
        $this->commandBus->send(
            new RegisterCustomer($request->get('name')),
            metadata: [
                // This is the Message Header that will be used for mapping tenant
                'tenant' => 'tenant_a'
            ]  
        );
        
        return new Response(200);        
    }
}
```

This way we are telling Ecotone, that we want to execute this Command in context of **tenant\_a**.

## Defining Message Handlers

We define Message Handler the same way we would do it for Single Tenant application. Yet we need to be aware that we need to make use of correct Connection for the job.

```php
final readonly class CustomerService
{
    #[CommandHandler]
    public function handle(
        RegisterCustomer $command,
        // Injecting Connection for current Tenant
        #[MultiTenantConnection] Connection $connection
    ): void
    {
        // store Customer
    }
}
```

By marking **Connection** with **MultiTenantConnection**, Ecotone will understand that it should inject Connection for Tenant in current context.
