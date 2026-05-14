---
description: DDD Aggregates PHP
---

# Aggregate Introduction

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

You have an `Order` model. The rule "an order can only be paid once" lives in the `OrderController::pay()` action. There's another check in the Stripe webhook listener. There's a third in the admin override controller. A new developer adds a fourth code path — a console command that updates the row directly — and skips the check. Production ships double-paid orders.

The root cause: anyone in the codebase can write `$order->status = 'paid'` or `$order->setStatus('paid')`. There's no enforcement boundary. The invariant is whatever the most-careful caller remembered to check.

## How Ecotone Solves It

An **Aggregate** is a class where the only way to mutate state is through `#[CommandHandler]` methods. Those methods are the rules. There's no `setStatus()` because the rule is: "to mark an order as paid, send a `MarkOrderAsPaid` command, and the aggregate decides whether that's allowed." The webhook, the controller, the console command — they all go through the same path. The invariant lives in one method and cannot be bypassed.

Ecotone handles loading the aggregate from your repository, routing the command to the right method, and saving the result. You write the business logic; the framework handles the plumbing.

---

This chapter covers the basics of implementing an [Aggregate](../../message-driven-php-introduction.md#aggregates). We will use Command Handlers, so first read the [External Command Handler](../external-command-handlers/) section to understand how Commands are sent and handled.

## Aggregate Command Handlers

Working with Aggregate Command Handlers is the same as with [External Command Handlers](../external-command-handlers/) — mark a method with `#[CommandHandler]` and Ecotone registers it. The difference is that Aggregate handlers are *instance* methods (or static factory methods), and Ecotone takes care of fetching the aggregate, calling the method, and saving the result:

```php
// Without Ecotone — the same three lines repeated in every handler:
$product = $this->repository->getById($command->id());
$product->changePrice($command->getPriceAmount());
$this->repository->save($product);
```

With Ecotone, you write only the middle line — declared as a method on the aggregate itself.

```php
#[Aggregate]
class Product
{
    #[Identifier]
    private string $productId;
```

By providing the `#[Identifier]` attribute, you tell Ecotone which property identifies this Aggregate (an Entity in Symfony/Doctrine, a Model in Laravel). Ecotone uses it to fetch the aggregate before each command.

When you send a Command, Ecotone reads the property with the matching name from the Command and uses it to load the Aggregate.

```php
class ChangePriceCommand
{
    private string $productId; // same property name as Aggregate's Identifier
    private Money $priceAmount;
```

{% hint style="success" %}
For more advanced scenarios — mapping by expression, multiple identifiers, or computed identifiers — see [Identifier Mapping](../identifier-mapping.md).
{% endhint %}

Once the identifier resolves, Ecotone uses the configured Repository to fetch the aggregate, call the method, and save the result.

{% hint style="success" %}
You don't need to implement a Repository yourself. Ecotone provides built-in [Event Sourcing Repositories](../../event-sourcing/), [Document Store Repositories](../../../messaging/document-store.md#storing-aggregates-in-your-document-store), and integrations with [Doctrine ORM](../../../modules/symfony/doctrine-orm.md) and [Eloquent](../../../modules/laravel/eloquent.md). See the [Repository section](../repository/) for details.
{% endhint %}

## State-Stored Aggregate

An Aggregate is a regular object that owns state and the methods that change it. Instead of public setters, it exposes `#[CommandHandler]` methods — each one is a business operation that enforces its own invariants:

```php
#[Aggregate] // 1
class Order
{
    #[Identifier] // 2
    private string $orderId;

    private OrderStatus $status;
    private Money $amount;

    private function __construct(string $orderId, Money $amount)
    {
        $this->orderId = $orderId;
        $this->amount = $amount;
        $this->status = OrderStatus::PLACED;
    }

    #[CommandHandler] // 3
    public static function place(PlaceOrder $command): self
    {
        if ($command->amount->isNegativeOrZero()) {
            throw new InvalidOrderAmount();
        }

        return new self($command->orderId, $command->amount);
    }

    #[CommandHandler] // 4
    public function pay(MarkOrderAsPaid $command, PaymentGateway $gateway): void
    {
        if ($this->status === OrderStatus::PAID) {
            throw new OrderAlreadyPaid($this->orderId);
        }
        if ($this->status === OrderStatus::CANCELLED) {
            throw new CannotPayCancelledOrder();
        }

        $gateway->charge($this->amount, $command->paymentMethod);
        $this->status = OrderStatus::PAID;
    }
}
```

1. `#[Aggregate]` tells Ecotone this class is an Aggregate Root.
2. `#[Identifier]` marks the external reference point. Ecotone uses it to load the aggregate when a command arrives.
3. A `#[CommandHandler]` on a *static* method is a **factory** — it must return a new aggregate instance. The constructor stays private; the factory is the only way to create the aggregate.
4. A `#[CommandHandler]` on an *instance* method is a **business operation**. The aggregate is fetched, the method runs, and the result is saved. Note there's no `setStatus()` — the only path from `PLACED` to `PAID` is through `pay()`, which enforces the rule that an order cannot be paid twice.

The webhook listener, the admin controller, the retry job — they all do the same thing: send a `MarkOrderAsPaid` command. The rule lives once, in `Order::pay()`, and the rest of the codebase cannot work around it.
