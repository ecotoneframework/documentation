# Message Obfuscation

One of the methods to protect your data is to obfuscate messages which are sent with queue using secure key you've defined.

### Obfuscate Channel

Channel can be protected in general which means every message will be obfuscated. In order to do that, you have to provide `ChannelProtectionConfiguration`.&#x20;

```php
use Ecotone\DataProtection\Configuration\ChannelProtectionConfiguration;

class DataProtection
{
    #[ServiceContext]
    public function paymentChannelProtectionConfiguration(): ChannelProtectionConfiguration
    {
        return ChannelProtectionConfiguration::create(
            channelName: 'payment', // define which channel needs protection
            encryptionKey: 'primary-key' // define which key should be used
            isPayloadSensitive: true, // // define if payload of messages sent by `payment` channel whould be obfuscated
            sensitiveHeaders: ['credit-card-number', 'iban'], // if occurs, those headers will be obfuscated. Ecotone won't add them.
        );
    }

    // payloads are considered sensitive by default
    // if not defined, channel will be secured with default key
    #[ServiceContext]
    public function dummyChannelProtectionConfiguration(): ChannelProtectionConfiguration
    {
        return ChannelProtectionConfiguration::create(
            channelName: 'dummy',
            sensitiveHeaders: ['foo', 'bar'],
        );
    }
}
```

### Obfuscate Message

You can also obfuscate single message by using Data Protection attributes directly in your class.

```php
use Ecotone\DataProtection\Attribute\Sensitive;
use Ecotone\DataProtection\Attribute\WithEncryptionKey;
use Ecotone\DataProtection\Attribute\WithSensitiveHeader;

#[Sensitive] // tells Ecotone that this message is sensitive
#[WithEncryptionKey('secondary-key')] // optional, tells Ecotone which key should be used. If not defined, Ecotone will use default key.
#[WithSensitiveHeader('iban')] // optional (repeatable), tells Ecotone that if message is sent with `foo` header, its value should be encrypted as well
readonly class ChargeCreditCard
{
    public function __construct(
        // ...
    ) {
    }
}
```

### Obfuscate Endpoint

Message obfuscation can be also defined at endpoint. Data Protection attributes can be used with `#[Payload]` or `#[Header]` .

```php
use Ecotone\DataProtection\Attribute\Sensitive;
use Ecotone\DataProtection\Attribute\WithEncryptionKey;
use Ecotone\DataProtection\Attribute\WithSensitiveHeader;

#[Asynchronous('payment')]
class PaymentProcessor
{
    #[CommandHandler(endpointId: 'payment.chargeCreditCard')]
    #[WithSensitiveHeader('name')] // headers unused in method can still be sent as sensitive
    public function chargeCreditCard(
        #[Sensitive] #[WithEncryptionKey('secondary-key')] ChargeCreditCard $message, // tells Ecotone that message handled by this endpoint is sensitive 
        #[Sensitive] #[Header('iban')] string $iban,
        // ...
    ): void {
        // ...
    }
}
```

{% hint style="warning" %}
Message configuration will **always** have higher precedence.
{% endhint %}

In following example, `ChargeCreditCard` command and `iban` header will be secured with `secondary-key` despite `payment` channel uses `primary-key`  and endpoint is defined with `another-key`.

```php
use Ecotone\DataProtection\Attribute\Sensitive;
use Ecotone\DataProtection\Attribute\WithEncryptionKey;
use Ecotone\DataProtection\Attribute\WithSensitiveHeader;

#[Sensitive]
#[WithEncryptionKey('secondary-key')]
#[WithSensitiveHeader('iban')]
readonly class ChargeCreditCard
{
    public function __construct(
        // ...
    ) {
    }
}

#[Asynchronous('payment')]
class PaymentProcessor
{
    #[CommandHandler(endpointId: 'payment.chargeCreditCard')]
    public function chargeCreditCard(
        #[Sensitive] #[WithEncryptionKey('another-key')] ChargeCreditCard $message,
        #[Sensitive] #[Header('iban')] string $iban,
        // ...
    ): void {
        // ...
    }
}

class DataProtection
{
    #[ServiceContext]
    public function paymentChannelProtectionConfiguration(): ChannelProtectionConfiguration
    {
        return ChannelProtectionConfiguration::create(
            channelName: 'payment',
            encryptionKey: 'primary-key'
            isPayloadSensitive: true,
            sensitiveHeaders: ['credit-card-number'],
        );
    }
}
```
