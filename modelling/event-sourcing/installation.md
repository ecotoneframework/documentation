# Installation

Ecotone comes with full automation for setting up Event Sourcing for us. This we can we really easily roll out new features with Event Sourcing with just minimal or none setup at all.&#x20;

### Install Event Sourcing Support

Before we will start, let's first install Event Sourcing module, which will provide us with all required components:

```php
composer require ecotone/pdo-event-sourcing
```

We need to configure [DBAL Support](../../modules/dbal-support.md) in order to make use of it.

Ecotone PDO Event Sourcing does provide support for three databases:

* PostgreSQL
* MySQL
* MariaDB

### Install Inbuilt Serialization Support

Ecotone provides inbuilt functionality to serialize your Events, which can be customized in case of need. This makes Ecotone take care of Event Serialization/Deserialization, and allows us to focus on the business side of the code.&#x20;

We can take over this process and set up our own [Serialization](../../messaging/conversion/conversion/), however [Ecotone JMS Converter](../../modules/jms-converter.md) can fully do it for us, so we can simply focus on the business side of the code.  To make it happen all we need to do, is to install JMS Package and we are ready to go:

```php
composer require ecotone/jms-converter
```
