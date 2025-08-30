---
description: Learn how to build long-running workflows that remember state using Sagas
---

# Sagas: Workflows That Remember

While [connecting handlers with channels](connecting-handlers-with-channels.md) works great for simple workflows, some business processes need to **remember what happened before**. This is where Sagas come in.

## When Do You Need Sagas?

Use Sagas when your workflow needs to:

🌳 **Branch and merge**: Split into multiple paths that later combine ⏳ **Wait for humans**: Pause for manual approval or external actions 📊 **Track progress**: Monitor long-running processes (hours, days, weeks) 🔄 **Handle complexity**: Make decisions based on accumulated state

**Examples:**

* Order processing (payment → shipping → delivery)
* Loan approval (application → verification → decision)
* User onboarding (signup → verification → welcome)

{% hint style="success" %}
**Think of Sagas as**: A workflow coordinator that remembers what happened and decides what to do next based on that history.
{% endhint %}

{% hint style="info" %}
**Prerequisites**: Familiarity with [connecting handlers](connecting-handlers-with-channels.md) and [aggregates](../command-handling/state-stored-aggregate/) will help you understand Sagas better.
{% endhint %}

## Creating Your First Saga

A Saga is like a persistent coordinator that remembers its state between events. Let's build an order processing saga step by step.

### Step 1: Define the Saga Structure

```php
#[Saga]
final class OrderProcess
{
    private function __construct(
        #[Identifier] private string $orderId,  // 👈 Unique ID to find this saga
        private OrderStatus $status,            // 👈 Current state
        private bool $isPaid = false,           // 👈 Remember payment status
        private bool $isShipped = false,        // 👈 Remember shipping status
    ) {}
}
```

**Key parts:**

* `#[Saga]` - Tells Ecotone this is a stateful workflow
* `#[Identifier]` - Unique ID to find and update this specific saga instance
* Private properties - The state that gets remembered between events

### Step 2: Start the Saga

Sagas begin when something important happens (usually an event):

```php
#[Saga]
final class OrderProcess
{
    // ... constructor ...

    #[EventHandler]  // 👈 This starts the saga
    public static function startWhen(OrderWasPlaced $event): self
    {
        return new self(
            orderId: $event->orderId,
            status: OrderStatus::PLACED
        );
    }
}
```

**What happens:**

1. `OrderWasPlaced` event occurs
2. Ecotone creates a new `OrderProcess` saga instance
3. The saga is saved to storage (database, etc.)
4. Now it can react to future events for this order

### Storage

Sagas are automatically stored using [Repositories](../command-handling/repository/repository.md). You can use Doctrine ORM, Eloquent, or Ecotone's Document Store - no extra configuration needed!

{% hint style="success" %}
**Saga vs Aggregate**: Both can handle events and commands, but use Sagas for **business processes** (workflows) and Aggregates for **business rules** (data consistency).
{% endhint %}

## Step 3: React to Events and Take Actions

Now let's make the saga react to events and coordinate the workflow:

### Triggering Commands

When payment succeeds, we want to start shipping:

```php
#[Saga]
final class OrderProcess
{
    // ... previous methods ...

    #[EventHandler(outputChannelName: "shipOrder")]  // 👈 Send to shipping
    public function whenPaymentSucceeded(PaymentWasSuccessful $event): ShipOrder
    {
        // Update saga state
        $this->isPaid = true;
        $this->status = OrderStatus::READY_TO_SHIP;

        // Return command to trigger shipping
        return new ShipOrder($this->orderId);
    }
}
```

**The flow:**

1. `PaymentWasSuccessful` event arrives
2. Saga updates its internal state
3. Saga returns `ShipOrder` command
4. Command goes to `shipOrder` channel
5. Shipping handler processes the order

### Alternative: Using Command Bus

You can also send commands directly:

```php
#[EventHandler]
public function whenPaymentSucceeded(PaymentWasSuccessful $event, CommandBus $commandBus): void
{
    $this->isPaid = true;
    $this->status = OrderStatus::READY_TO_SHIP;

    $commandBus->send(new ShipOrder($this->orderId));
}
```

{% hint style="info" %}
**Timing difference**:

* `outputChannelName`: Saga state saved first, then command is sent
* `CommandBus`: Command sent first, then saga state saved
{% endhint %}

## Step 4: Publishing Events and Timeouts

Sagas can also publish events to trigger other parts of your system or set up timeouts.

### Publishing Events

```php
#[Saga]
final class OrderProcess
{
    use WithEvents;  // 👈 Enables event publishing

    private function __construct(
        #[Identifier] private string $orderId,
        private OrderStatus $status,
        private bool $isPaid = false,
    ) {
        // Publish event when saga starts
        $this->recordThat(new OrderProcessStarted($this->orderId));
    }

    #[EventHandler]
    public function whenShippingCompleted(OrderWasShipped $event): void
    {
        $this->isShipped = true;
        $this->status = OrderStatus::COMPLETED;

        // Publish completion event
        $this->recordThat(new OrderProcessCompleted($this->orderId));
    }
}
```

### Setting Up Timeouts

Cancel orders that aren't paid within 24 hours:

```php
#[Saga]
final class OrderProcess
{
    // ... other methods ...

    #[Delayed(new TimeSpan(days: 1))]  // 👈 Wait 24 hours
    #[Asynchronous('async')]
    #[EventHandler]
    public function cancelUnpaidOrder(OrderProcessStarted $event): void
    {
        if (!$this->isPaid) {
            $this->status = OrderStatus::CANCELLED;
            $this->recordThat(new OrderWasCancelled($this->orderId, 'Payment timeout'));
        }
    }
}
```

**Timeline:**

* ⏰ **T+0**: Order placed, saga starts, timeout scheduled
* ⏰ **T+24h**: If still unpaid, automatically cancel

## Advanced Patterns

### Conditional Event Handling

Sometimes you only want to handle events if the saga already exists. Use `dropMessageOnNotFound`:

```php
#[Saga]
class CustomerPromotion
{
    private function __construct(
        #[Identifier] private string $customerId,
        private bool $hasGreatCustomerBadge = false
    ) {}

    #[EventHandler]
    public static function startWhen(ReceivedGreatCustomerBadge $event): self
    {
        return new self($event->customerId, hasGreatCustomerBadge: true);
    }

    #[EventHandler(dropMessageOnNotFound: true)]  // 👈 Only if saga exists
    public function whenOrderPlaced(OrderWasPlaced $event, CommandBus $commandBus): void
    {
        // Only send promo if customer has the badge (saga exists)
        $commandBus->send(new SendPromotionCode($this->customerId));
    }
}
```

**How it works:**

* ✅ If saga exists: Event is processed
* ❌ If saga doesn't exist: Event is ignored (dropped)
* 🎯 **Use case**: Features that depend on previous conditions

### Querying Saga State

Expose saga state to your application (great for status pages, dashboards):

```php
#[Saga]
final class OrderProcess
{
    // ... other methods ...

    #[QueryHandler("order.getStatus")]
    public function getStatus(): array
    {
        return [
            'orderId' => $this->orderId,
            'status' => $this->status->value,
            'isPaid' => $this->isPaid,
            'isShipped' => $this->isShipped,
        ];
    }

    #[QueryHandler("order.getProgress")]
    public function getProgress(): int
    {
        $steps = 0;
        if ($this->isPaid) $steps++;
        if ($this->isShipped) $steps++;

        return ($steps / 2) * 100; // Progress percentage
    }
}
```

**Using in your controllers:**

```php
class OrderController
{
    public function __construct(private QueryBus $queryBus) {}

    public function getOrderStatus(string $orderId): JsonResponse
    {
        $status = $this->queryBus->sendWithRouting(
            'order.getStatus',
            metadata: ['aggregate.id' => $orderId]  // 👈 Target specific saga
        );

        return new JsonResponse($status);
    }
}
```

**Perfect for:**

* Order tracking pages
* Progress indicators
* Admin dashboards
* Customer support tools

### Handling Unordered Events

Real-world events don't always arrive in order. Ecotone handles this elegantly with **method redirection**:

```php
#[Saga]
class OrderFulfillment
{
    private function __construct(
        #[Identifier] private string $orderId,
        private bool $isStarted = false
    ) {}

    // 👇 If saga doesn't exist, this creates it
    #[EventHandler]
    public static function startByOrderPlaced(OrderWasPlaced $event): self
    {
        return new self($event->orderId, isStarted: true);
    }

    // 👇 If saga exists, this handles the event
    #[EventHandler]
    public function whenOrderWasPlaced(OrderWasPlaced $event): void
    {
        // Handle additional order placed logic
        // (maybe it's a duplicate event, or additional items)
    }
}
```

**How Ecotone decides:**

* 🎯 **Same event, different behavior** based on saga state
* 🆕 **Saga doesn't exist** → Calls static factory method
* ✅ **Saga exists** → Calls action method

**Benefits:**

* No complex if/else logic in your code
* Handles event ordering issues automatically
* Clean separation of initialization vs. processing logic

{% hint style="success" %}
**Pro tip**: This pattern is perfect for handling duplicate events or events that can arrive at different workflow stages.
{% endhint %}

## Saga Identification and Correlation

### Finding the Right Saga Instance

Every event/command needs to find the correct saga instance. Ecotone uses [Identifier Mapping](../command-handling/identifier-mapping.md) for this:

```php
// Event with orderId property
class PaymentWasSuccessful
{
    public function __construct(public string $orderId) {}
}

// Saga with orderId identifier
#[Saga]
class OrderProcess
{
    public function __construct(
        #[Identifier] private string $orderId  // 👈 Matches event property
    ) {}

    #[EventHandler]
    public function whenPaymentSucceeded(PaymentWasSuccessful $event): void
    {
        // Ecotone automatically finds the saga with matching orderId
    }
}
```

### Using Correlation IDs

For complex workflows that branch and merge, use correlation IDs:

```php
#[Saga]
class OrderProcess
{
    public function __construct(
        #[Identifier] private string $correlationId  // 👈 Tracks across branches
    ) {}

    #[EventHandler]
    public static function startWhen(OrderWasPlaced $event): self
    {
        // Use correlation ID from message headers
        return new self($event->getHeaders()['correlationId']);
    }
}
```

**Benefits of correlation IDs:**

* Track workflows across multiple services
* Handle branching and merging flows
* Automatic propagation through message chains

{% hint style="info" %}
**Correlation IDs** are automatically propagated between messages, making them perfect for complex workflows that span multiple services or branches.
{% endhint %}

## Testing Sagas with Ecotone Lite

Testing sagas is essential for ensuring your stateful workflows behave correctly. Ecotone Lite makes saga testing straightforward and comprehensive.

### Setting Up Saga Tests

```php
use Ecotone\Lite\EcotoneLite;
use PHPUnit\Framework\TestCase;

class OrderProcessSagaTest extends TestCase
{
    private EcotoneLite $ecotoneLite;

    protected function setUp(): void
    {
        $this->ecotoneLite = EcotoneLite::bootstrapFlowTesting(
            [OrderProcess::class],
            [],
        );
    }

    public function test_saga_initialization(): void
    {
        // Act - Trigger saga creation
        $event = new OrderWasPlaced('order-123');
        $this->ecotoneLite->publishEvent($event);

        // Assert - Verify saga was created and can be queried
        $status = $this->ecotoneLite->sendQueryWithRouting(
            'order.getStatus',
            metadata: ['aggregate.id' => 'order-123']
        );

        $this->assertEquals('order-123', $status['orderId']);
        $this->assertEquals('PLACED', $status['status']);
        $this->assertFalse($status['isPaid']);
        $this->assertFalse($status['isShipped']);
    }
}
```

### Testing Saga State Changes

Test how sagas respond to events and update their state:

```php
public function test_payment_processing_updates_saga_state(): void
{
    // Arrange - Create saga
    $this->ecotoneLite->publishEvent(new OrderWasPlaced('order-123'));

    // Act - Process payment
    $this->ecotoneLite->publishEvent(new PaymentWasSuccessful('order-123'));

    // Assert - Verify state changed
    $status = $this->ecotoneLite->sendQueryWithRouting(
        'order.getStatus',
        metadata: ['aggregate.id' => 'order-123']
    );

    $this->assertTrue($status['isPaid']);
    $this->assertEquals('READY_TO_SHIP', $status['status']);
}
```

## Summary: When to Use Sagas

✅ **Use Sagas when you need to:**

* Remember state between events
* Coordinate long-running processes
* Handle branching/merging workflows
* Implement timeouts and cancellations
* Track progress of complex operations

❌ **Don't use Sagas for:**

* Simple linear workflows (use [handler chaining](connecting-handlers-with-channels.md))
* Stateless transformations

{% hint style="success" %}
**Key insight**: Sagas are workflow coordinators that remember what happened and decide what to do next. They're perfect for orchestrating complex business processes that span across time and multiple services.
{% endhint %}
