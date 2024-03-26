# Laravel

## Configuring Connections

While using Laravel we can reuse existing Connections (**config/database.php**).

```
return [
    'default' => 'tenant_a_connection',
    'connections' => [
        'tenant_a_connection' => [
            'url' => getenv('TENANT_A_DATABASE_URL')
        ],

        'tenant_b_connection' => [
            'url' => getenv('TENANT_B_DATABASE_URL')
        ],
    ]
];
```

And then we can set up Mapping:

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]
    public function multiTenantConfiguration(): MultiTenantConfiguration
    {
        return MultiTenantConfiguration::create(
            tenantHeaderName: 'tenant', // Message header, where to look for tenant
            tenantToConnectionMapping: [
                'tenant_a' => LaravelConnectionReference::create('tenant_a_connection'),
                'tenant_b' => LaravelConnectionReference::create('tenant_b_connection')
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
            new RegisterCustomer($request->input('name')),
            metadata: [
                // This is the Message Header that will be used for mapping tenant
                'tenant' => 'tenant_a'
            ]  
        );
        
        return new Response(200);        
    }
}
```

## Defining Message Handlers

We define Message Handler the same way we would do it for Single Tenant application. Yet we need to be aware that we need to make use of correct Entity Manager for the job.

```php
final readonly class CustomerService
{
    #[CommandHandler]
    public function handle(RegisterCustomer $command)
    {
        Customer::register($command)->save();
    }
}
```

Like we can see here, there is no code being aware of Multi-Tenancy here. This is because Ecotone will understand the current context and will switch default database to Tenant's database. \
This way Customer will be stored in correct DB.
