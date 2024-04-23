# Symfony and Doctrine ORM

## Configuration with Symfony

For Entity Manager while using Doctrine ORM, we can levarage existing Symfony configuration.\
We use then register separate Entity Manager per Tenant, as each has it's own unique connection (**doctrine.yaml**):

```yaml
doctrine:
  dbal:
    connections:
      tenant_a_connection:
        url: '%env(resolve:TENANT_A_DATABASE_URL)%'
        charset: UTF8
      tenant_b_connection:
        url: '%env(resolve:TENANT_B_DATABASE_URL)%'
        charset: UTF8
  orm:
    entity_managers:
      tenant_a_connection:
        connection: tenant_a_connection
        mappings:
          (Our mapping goes here)
      tenant_b_connection:
        connection: tenant_b_connection
        mappings:
          (Our mapping goes here)
```

and then we can set up Mapping

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function multiTenantConfiguration(): MultiTenantConfiguration
    {
        return MultiTenantConfiguration::create(
            tenantHeaderName: 'tenant',
            tenantToConnectionMapping: [
                'tenant_a' => SymfonyConnectionReference::createForManagerRegistry('tenant_a_connection'),
                'tenant_b' => SymfonyConnectionReference::createForManagerRegistry('tenant_b_connection')
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

## Accessing Tenant's Connection

To access current Tenant's Connection, we will be using Atribute:

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
        // do something
    }
}
```

By marking **Connection** with **MultiTenantConnection**, Ecotone will understand that it should inject Connection for Tenant in current context.

## Accessing Tenant's Object Manager

To access current Tenant's Object Manager, we will be using Atribute:

```php
final readonly class CustomerService
{
    #[CommandHandler]
    public function handle(
        RegisterCustomer $command,
        // Injecting ObjectManager for current Tenant
        #[MultiTenantObjectManager] ObjectManager $objectManager
    ): void
    {
        $objectManager->persist(Customer::register($command));
    }
}
```

By marking **ObjectManager** with **MultiTenantObjectManager**, Ecotone will understand that it should inject ObjectManager for Tenant in current context.
