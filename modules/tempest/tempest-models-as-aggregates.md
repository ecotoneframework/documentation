---
description: Using Tempest active-record models as Ecotone Aggregates
---

# Tempest Models as Aggregates

Ecotone comes with out-of-the-box integration for using Tempest's active-record models (those using the `IsDatabaseModel` trait) as [State-Stored Aggregates](../../modelling/command-handling/state-stored-aggregate/#state-stored-aggregate). This is the Tempest equivalent of [Eloquent on Laravel](../laravel/eloquent.md) and [Doctrine ORM on Symfony](../symfony/doctrine-orm.md).

## Your Models as Aggregates

Mark your model with the `#[Aggregate]` attribute and add Command Handlers:

```php
use Ecotone\Modelling\Attribute\Aggregate;
use Ecotone\Modelling\Attribute\CommandHandler;
use Ecotone\Modelling\Attribute\IdentifierMethod;
use Ecotone\Modelling\Attribute\QueryHandler;
use Tempest\Database\IsDatabaseModel;
use Tempest\Database\PrimaryKey;

#[Aggregate]
final class Product
{
    use IsDatabaseModel;

    public PrimaryKey $id;
    public string $name;
    public int $price;

    #[CommandHandler] // 1. factory method
    public static function register(RegisterProduct $command): self
    {
        $product = new self();
        $product->name = $command->name;
        $product->price = $command->price;
        $product->save(); // 3. Saving

        return $product;
    }

    #[CommandHandler('product.changePrice')] // 2. action method
    public function changePrice(ChangePrice $command): void
    {
        $this->price = $command->price;
    }

    #[QueryHandler('product.getPrice')]
    public function getPrice(): int
    {
        return $this->price;
    }

    #[IdentifierMethod('id')] // 4. expose the scalar identifier
    public function getId(): int
    {
        return $this->id->value;
    }
}
```

1. Calling the factory method:

```php
$id = $this->commandBus->send(new RegisterProduct('Milk', 100));
```

2. Calling the action method:

```php
$this->commandBus->sendWithRouting('product.changePrice', new ChangePrice(200), metadata: ['aggregate.id' => $id]);
```

3. Aggregates require state to be always valid. Tempest assigns the auto-increment `PrimaryKey` on `save()`, so call `save()` in the factory to obtain the identifier. If you generate identifiers outside the database, this step is not needed.
4. `#[IdentifierMethod]` exposes the scalar identifier Ecotone uses to load and route to the aggregate (Tempest stores it as a `PrimaryKey` value object).

{% hint style="success" %}
You can use routing for your Message Handlers or direct Message Classes — whatever works best in your context.
{% endhint %}

## Repository and Business Interface

Because a Tempest model is a state-stored Aggregate, Ecotone persists it automatically through the `TempestRepository` when a Command Handler returns or mutates it — no repository wiring is required.

You can additionally declare a [DBAL Business Interface](../../modelling/command-handling/business-interface/working-with-database/) (`#[DbalQuery]` / `#[DbalWrite]`) for read-side queries over the same connection.

## Recording Events from your Model

Tempest's ORM persists the model's properties to columns. Ecotone's generic `WithEvents` trait keeps recorded events in a private property, which Tempest would try to map to a `recordedEvents` column. Declare your own recorded-events property marked with Tempest's `#[Virtual]` attribute instead, and release the events with `#[AggregateEvents]`:

```php
use Ecotone\Modelling\Attribute\Aggregate;
use Ecotone\Modelling\Attribute\AggregateEvents;
use Tempest\Database\IsDatabaseModel;
use Tempest\Database\Virtual;

#[Aggregate]
final class Order
{
    use IsDatabaseModel;

    #[Virtual]
    private array $recordedEvents = [];

    private function recordThat(object $event): void
    {
        $this->recordedEvents[] = $event;
    }

    #[AggregateEvents]
    public function releaseEvents(): array
    {
        $events = $this->recordedEvents;
        $this->recordedEvents = [];

        return $events;
    }
}
```

With this in place, events recorded in your Command Handlers are published to Event Handlers after the aggregate is saved — `#[Virtual]` keeps the property out of the database mapping.
