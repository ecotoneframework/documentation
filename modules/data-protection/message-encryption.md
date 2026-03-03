---
description: Protect Messages using direct annotations
---

# Message Encryption

You can protect single message by using Data Protection attributes directly in your class.

Adding `#[Sensitive]` annotation to class will tell Ecotone to treat all attributes as sensitive.

```php
use Ecotone\DataProtection\Attribute\Sensitive;
use Ecotone\DataProtection\Attribute\WithEncryptionKey;

#[Sensitive] // tells Ecotone that this message is sensitive
#[WithEncryptionKey('secondary-key')] // optional, tells Ecotone which key should be used. If not defined, Ecotone will use default key.
readonly class ChargeCreditCard
{
    public function __construct(
        // ...
    ) {
    }
}
```

You can also add `#[Sensitive]`  annotation to specific properties, which will make Ecotone to protect only those.&#x20;

```php
use Ecotone\DataProtection\Attribute\Sensitive;
use Ecotone\DataProtection\Attribute\WithEncryptionKey;

#[Sensitive] // tells Ecotone that this message is sensitive
#[WithEncryptionKey('secondary-key')] // optional, tells Ecotone which key should be used. If not defined, Ecotone will use default key.
readonly class ChargeCreditCard
{
    public function __construct(
        public string $walletId,
        #[Sensitive] public IbanNumber $iban,
        // ...
    ) {
    }
}
```

`#[Sensitive]`  attribute can be used with any variable: scalar, class or enum.

{% hint style="info" %}
If you mark Event Sourcing Events as sensitive, they will be persisted with encrypted data.
{% endhint %}

```php
use Ecotone\DataProtection\Attribute\Sensitive;
use Ecotone\DataProtection\Attribute\WithEncryptionKey;

#[Sensitive]
#[NamedEvent('payment.credit-card-was-changed')]
readonly class CreditCardWasCharged
{
    public function __construct(
        public string $walletId,
        #[Sensitive] public IbanNumber $iban, // iban attribute of event payload will be encrypted in Event Store
        // ...
    ) {
    }
}
```

