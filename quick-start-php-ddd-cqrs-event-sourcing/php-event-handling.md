---
description: Event Handlers PHP
---

# Event Handling PHP

## Demo

{% embed url="https://github.com/ecotoneframework/quickstart-examples/tree/main/EventHandling" %}

## Read Blog Post

[Read more about Event Handling in PHP and Ecotone](https://blog.ecotone.tech/event-handling-in-php/)

## Code Example

Let's create `Event` _Order was placed_.

```php
class OrderWasPlaced
{
    private string $orderId;
    private string $productName;

    public function __construct(string $orderId, string $productName)
    {
        $this->orderId = $orderId;
        $this->productName = $productName;
    }

    public function getOrderId(): string
    {
        return $this->orderId;
    }

    public function getProductName(): string
    {
        return $this->productName;
    }
}
```

&#x20;And Event Handler that will be listening to the `OrderWasPlaced`.

```php
class NotificationService
{
    #[EventHandler]
    public function notifyAboutNewOrder(OrderWasPlaced $event) : void
    {
        echo $event->getProductName() . "\n";
    }
}
```

## Running The Example

```php
$eventBus->publish(new OrderWasPlaced(1, "Milk"));
```
