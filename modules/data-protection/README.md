---
description: Ecotone enables protection for data sent in messages.
icon: file-shield
---

# Data Protection

{% hint style="info" %}
## **This module is available as part of Ecotone Enterprise.**
{% endhint %}

Ecotone will help you to encrypt your messeges (Events or Commands) right before they leave and decrypt them when they are received by your domain. In other words, message will remain readable within the application but once they leave the system, secured key is required for reading.

## Installation

```bash
composer require ecotone/data-protection
```

## Configuration

Required `DataProtectionConfiguration` will let you to provide set of encryption keys used within the system.

```php
use Ecotone\DataProtection\Configuration\DataProtectionConfiguration;
use Ecotone\DataProtection\Encryption\Key;

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
Ecotone does not manage your encryption keys. They are used based on given configuration.
{% endhint %}
