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

With `#[Sensitive]`  attribute you can annotate whole class or specific properties, which will make Ecotone to protect only those. Attribute can be used with any type: scalar, class or enum.

```php
use Ecotone\DataProtection\Attribute\Sensitive;
use Ecotone\DataProtection\Attribute\WithEncryptionKey;

#[WithEncryptionKey('secondary-key')] // optional, tells Ecotone which key should be used. If not defined, Ecotone will use default key.
readonly class ChargeCreditCard
{
    public function __construct(
        public string $walletId,
        #[Sensitive] public IbanNumber $iban,// tells Ecotone that this property is sensitive
        // ...
    ) {
    }
}
```

{% hint style="info" %}
If you use #\[Sensitive] with Event Sourcing Events, they will be persisted with encrypted data.
{% endhint %}

```php
use Ecotone\DataProtection\Attribute\Sensitive;
use Ecotone\DataProtection\Attribute\WithEncryptionKey;

#[NamedEvent('payment.credit-card-was-changed')]
readonly class CreditCardWasCharged
{
    public function __construct(
        public string $walletId,
        #[Sensitive] public IbanNumber $iban, // event payload will contain encrypted value for iban in Event Store
        // ...
    ) {
    }
}
```

### Custom Converters

Data Protection extends standard conversion and uses actual property names. If you are using custom converters for your messages, you may change name of property you want to be protected. In that case, you can pass custom name with `#[Sensitive]` attribute to tell Ecotone how to handle data properly.

```php
use Ecotone\DataProtection\Attribute\Sensitive;
use Ecotone\Messaging\Attribute\Converter;

readonly class ChargeCreditCard
{
    public function __construct(
        public string $walletId,
        #[Sensitive('iban-number')] public IbanNumber $iban, // passing custom name works only with properties
    ) {
    }
}

class ChargeCreditCardConverter
{
    #[Converter]
    public function convertFrom(ChargeCreditCard $object): array
    {
        return [
            'walletId' => $object->walletId,
            'iban-number' => $object->iban->number, // Ecotone knows to look for value under `iban-number`
        ];
    }

    #[Converter]
    public function convertTo(array $payload): ChargeCreditCard
    {
        return new ChargeCreditCard(
            walletId: $payload['walletId'],
            iban: new IbanNumber($payload['iban-number']),
        );
    }
}

```
