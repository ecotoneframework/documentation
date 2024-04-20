# Doctrine ORM

Ecotone comes with out of the box Doctrine ORM support for Aggregates.&#x20;

## Enable Doctrine ORM

First install Dbal Module with [Manager Registry Connection](../dbal-support.md#using-manager-registry)[.](symfony-database-connection-dbal-module.md#using-manager-registry)

```php
class EcotoneConfiguration
{
    #[ServiceContext]
    public function getDbalConfiguration(): DbalConfiguration
    {
        return DbalConfiguration::createWithDefaults()
            ->withDoctrineORMRepositories(true);
    }
    
    # Configuration from Manager Registry Connection
    #[ServiceContext]
    public function getManagerRegistryConfiguration()
    {
        return SymfonyConnectionReference::defaultManagerRegistry('some_connection');
    }
}
```

### Example

Then you mark your Entities with Aggregate attribute and set up Command Handlers.

```php
#[Aggregate]
class User
{
    use WithEvents;

    #[AggregateIdentifier]
    private string $userId;
    private string $name;
    private bool $isActive;

    private function __construct(string $userId, string $name)
    {
        $this->userId = $userId;
        $this->name = $name;
        $this->isActive = false;
        
        $this-recordThat(new UserRegistered($userId)); // 3. Event Publishing
    }

    #[CommandHandler] // 1. Factory method register
    public static function register(RegisterUser $command): static
    {
        return new static(Uuid::uuid4()->toString(), $command->name);
    }

    #[CommandHandler("activate")] // 2. Action method "activate"
    public function activate(): void
    {
        $this->isActive = true;
    }
}
```

1. Calling factory method **"register":**

```php
$this->commandBus->send(new RegisterUser('Johny'));
```

2. Calling action method **"activate":**

```php
$this->commandBus->sendWithRouting("activate", metadata: ["aggregate.id" => $id]);
```

3. By importing trait `WithEvents` we can publish events from our Aggregate using `recordThat` method.

{% hint style="success" %}
In case of Ecotone you may use routing for your Message Handlers or direct Message Classes. It's up to you to decide whatever works best in your context.
{% endhint %}

### Flushing Doctrine ORM Changes

Ecotone will take care of object flush and clearing your object manager. This way we don't need to bother about integration part of the code. If you want take over the process this can be disabled via DbalConfiguration.

```php
final readonly class EcotoneConfiguration
{
    #[ServiceContext]   
    public function dbalConfiguration()
    {
        return DbalConfiguration::createWithDefaults()
                ->withClearAndFlushObjectManagerOnCommandBus(false)
                ->withClearAndFlushObjectManagerOnAsynchronousEndpoints(false);
    }
}
```

### Auto-Incremented Identifier

When using Doctrine ORM with auto incremented identifiers like sequences, identifier is not available from the beginning, yet it's assigned at later stage by Doctrine ORM itself. \
Ecotone works with auto-incremental identifiers, yet they must be initialized with null to state that:

```php
#[ORM\Entity]
#[ORM\Table(name: 'persons')]
#[Aggregate]
class Person
{
    #[ORM\Id()]
    #[ORM\Column(name: 'person_id', type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'SEQUENCE')]
    #[ORM\SequenceGenerator(sequenceName: 'person_id_seq')]
    #[Identifier]
    private ?int $personId = null;
```

### Enable Repository for given Aggregates

To enable Doctrine ORM for given set of Aggregates use DbalConfiguration:

```php
class EcotoneConfiguration
{
    #[ServiceContext]
    public function getDbalConfiguration(): DbalConfiguration
    {
        return DbalConfiguration::createWithDefaults()
            ->withDoctrineORMRepositories(
                true,
                [Article::class]
            );
    }
```

### Demo

Read more about integration in [following blog post](https://blog.ecotone.tech/build-symfony-application-with-ease-using-ecotone/).
