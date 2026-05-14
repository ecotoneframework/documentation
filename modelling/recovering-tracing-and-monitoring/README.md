---
description: Recovering, tracing, and monitoring message-driven applications
---

# Recovering, Tracing and Monitoring

A consumer crashes mid-handler. The Stripe webhook arrives twice. A `notify-customer` job fails forever and you can't tell why. Two workers race on the same aggregate and one's update gets silently overwritten. This section is the toolbox for everything that goes wrong after a message is dispatched.

Ecotone provides solutions across three concerns:

* **Self-healing** ([Instant and Delayed Retries](resiliency/retries.md), [Concurrency / Locking](resiliency/concurrency-handling.md), [Isolation of failures](message-handling-isolation.md))
* **Data Consistency** ([Resilient Message Sending](resiliency/resilient-sending.md), [Outbox pattern](resiliency/outbox-pattern.md), [Message Deduplication](resiliency/idempotent-consumer-deduplication.md))
* **Recovery & Visibility** ([Dead Letter](resiliency/error-channel-and-dead-letter/), [Tracing](../../modules/opentelemetry-tracing-and-metrics/), [Monitoring](ecotone-pulse-service-dashboard.md))

{% hint style="success" %}
To find out more about different use-cases, read related section about [Handling Failures in Workflows](../business-workflows/handling-failures.md).
{% endhint %}

## Materials

### Demo implementation

* [Error Handling with delayed retries and storing in DLQ](https://github.com/ecotoneframework/quickstart-examples/tree/main/ErrorHandling)

### Links

* [Async Failure Recovery: Queue vs Streaming Channel Strategies](https://blog.ecotone.tech/async-failure-recovery-queue-vs-streaming-channel-strategies/) {Article]
* [Read in depth material about resiliency in Messaging Systems using Ecotone](https://blog.ecotone.tech/building-reactive-message-driven-systems-in-php/) \[Article]
* [Resilient Messaging with Laravel](https://blog.ecotone.tech/ddd-and-messaging-with-laravel-and-ecotone/) \[Article]
* [Making your application stable with Outbox Pattern](https://blog.ecotone.tech/implementing-outbox-pattern-in-php-symfony-laravel-ecotone/) \[Article]
* [Handling asynchronous errors](https://blog.ecotone.tech/working-with-asynchronous-failures-in-php/) \[Article]
