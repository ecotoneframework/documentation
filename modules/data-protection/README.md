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

When defining sensitive data in your message, you can tell Ecotone whether it contains sensitive payload or specify sensitive headers.
