# Recovering, Tracing and Monitoring

To keep the system reliable and resilient it's important to handle errors with grace. \
Ecotone provides solutions to handle failures within the system that helps in:

* Self-healing the application ([Instant and Delayed Message Handling Retries](resiliency/retries.md), [Locking](resiliency/concurrency-handling.md), [Isolation of failures](message-handling-isolation.md))
* Ensuring Data Consistency ([Resilient Message Sending](resiliency/resilient-sending.md), [Outbox pattern](resiliency/outbox-pattern.md), [Message Deduplication](resiliency/idempotent-consumer-deduplication.md))
* Recovering ([Dead Letter](resiliency/error-channel-and-dead-letter.md), [Tracing](../../modules/opentelemetry-tracing-and-metrics.md), [Monitoring](ecotone-pulse-service-dashboard.md))

{% hint style="success" %}
To find out more about different use-cases, read related section about [Handling Failures in Workflows](../../messaging/workflows/handling-failures.md).
{% endhint %}

## Materials

### Demo implementation

* [Error Handling with delayed retries and storing in DLQ](https://github.com/ecotoneframework/quickstart-examples/tree/main/ErrorHandling)

### Links

* [Read in depth material about resiliency in Messaging Systems using Ecotone](https://blog.ecotone.tech/building-reactive-message-driven-systems-in-php/) \[Article]
* [Resilient Messaging with Laravel](https://blog.ecotone.tech/ddd-and-messaging-with-laravel-and-ecotone/) \[Article]
* [Making your application stable with Outbox Pattern](https://blog.ecotone.tech/implementing-outbox-pattern-in-php-symfony-laravel-ecotone/) \[Article]
* [Handling asynchronous errors](https://blog.ecotone.tech/working-with-asynchronous-failures-in-php/) \[Article]
