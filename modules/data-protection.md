---
description: >-
  Ecotone enables protection for data sent outside of the application (e.g.
  using RabbitMQ) by obfuscating messages' payload and headers.
icon: file-shield
---

# Data Protection

{% hint style="info" %}
## **This module is available as part of Ecotone Enterprise.**
{% endhint %}

Ecotone will encrypt you messeges (Events or Commands) right before they are sent to queue and decrypt them when they are received. In other words, message will remain readable within the application but once they leave the system, secured key is required for reading.

## Installation

```bash
composer require ecotone/data-protection
```

## Configuration

Required `DataProtectionConfiguration` will let you to provide set of encryption keys used within the system.

```php
use Defuse\Crypto\Key;
use Ecotone\DataProtection\Configuration\DataProtectionConfiguration;

class DataProtection
{
    #[ServiceContext]
    public function dataProtectionConfiguration(): DataProtectionConfiguration
    {
        return DataProtectionConfiguration::create(name: 'primary-key', key: Key::loadFromAsciiSafeString(...)) // first key will be set as default
            ->withKey(name: 'secondary-key', key: Key::loadFromAsciiSafeString(...))
            ->withKey(name: 'default-key', key: Key::loadFromAsciiSafeString(...), asDefault: true) // overwrite default key passing `asDefault: true`
        ;
    }
}
```

{% hint style="warning" %}
Ecotone does not manage encryption keys
{% endhint %}

When defining sensitive data in your message, you can tell Ecotone whether it contains sensitive payload or specify which headers should be obfuscated.

### Obfuscate Channel

To obfuscate channel in general, you can provide `ChannelProtectionConfiguration`.&#x20;

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

To obfuscate single message, you can use Data Protection attributes directly in your messages.

```php
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

Message obfuscation can be also defined at endpoint.

```php
#[Asynchronous('payment')]
class PaymentProcessor
{
    #[Sensitive] // tells Ecotone that message handled by this endpoint is sensitive 
    #[WithSensitiveHeader('iban')] // optional
    #[CommandHandler(endpointId: 'payment.chargeCreditCard')]
    public function chargeCreditCard(
        ChargeCreditCard $message,
        // ...
    ): void {
        // ...
    }
}
```

Data protection can be also defined in parameters

```php
#[Asynchronous('payment')]
class PaymentProcessor
{
    #[CommandHandler(endpointId: 'payment.chargeCreditCard')]
    public function chargeCreditCard(
        #[Sensitive] ChargeCreditCard $message,
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

In following example, `ChargeCreditCard` command and `iban` header will be secured with `secondary-key` despite `payment` channel uses `primary-key`.&#x20;

```php
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
        #[Sensitive] #[WithEncryptionKey('another-key')] ChargeCreditCard $message, // encryption key can be defined with payload parameter
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
