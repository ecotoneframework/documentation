---
description: Asynchronous PHP
---

# Asynchronous Handling and Scheduling

{% hint style="info" %}
Works with: **Laravel**, **Symfony**, and **Standalone PHP**
{% endhint %}

## The Problem

You added async processing, but now you can't tell which messages are stuck, which failed silently, and which will retry forever. Going async required touching every handler — adding queue configuration, serialization logic, and retry strategies individually.

## How Ecotone Solves It

Ecotone makes any handler async with a single `#[Asynchronous]` attribute. Retries, error handling, and dead letter are configured at the channel level, not per handler. Switch between synchronous and asynchronous execution without changing your business code.

---

{% content-ref url="asynchronous-message-handlers.md" %}
[asynchronous-message-handlers.md](asynchronous-message-handlers.md)
{% endcontent-ref %}

{% content-ref url="asynchronous-message-bus-gateways.md" %}
[asynchronous-message-bus-gateways.md](asynchronous-message-bus-gateways.md)
{% endcontent-ref %}

{% content-ref url="scheduling.md" %}
[scheduling.md](scheduling.md)
{% endcontent-ref %}

{% content-ref url="dynamic-message-channels.md" %}
[dynamic-message-channels.md](dynamic-message-channels.md)
{% endcontent-ref %}

## Materials

### Demo implementation

You may find demo implementation [here](https://github.com/ecotoneframework/quickstart-examples/tree/main/OutboxPattern).

### Links

* [Queues and Streaming Channels Architecture](https://blog.ecotone.tech/async-failure-recovery-queue-vs-streaming-channel-strategies/) \[Article]
* [Asynchronous processing with zero configuration](https://blog.ecotone.tech/message-channels-zero-configuration-async-processing/) \[Article]
* [Asynchronous PHP To Support Stability Of Your Application](https://blog.ecotone.tech/asynchronous-php/) \[Article]
* [Asynchronous Messaging in Laravel](https://blog.ecotone.tech/ddd-and-messaging-with-laravel-and-ecotone/) \[Article]
* [Testing Asynchronous Messaging](../testing-support/testing-asynchronous-messaging.md) \[Documentation]
* [Testing Asynchronous Message Driven Architecture](https://blog.ecotone.tech/testing-messaging-architecture-in-php/) \[Article]
