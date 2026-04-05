---
description: Protect Messages using channel encryption
---

# Channel Encryption

Channel can be protected in general which means every message sent with it will be encrypted. In order to do that, you have to provide `ChannelProtectionConfiguration`.&#x20;

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
