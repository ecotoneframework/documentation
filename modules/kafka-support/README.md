---
description: Kafka integration for high-throughput event streaming in PHP
---

# Kafka Support

Native integration with [Apache Kafka](https://kafka.apache.org/) for high-throughput event streaming, using Ecotone's attribute-driven programming model. No separate producer/consumer boilerplate -- use `#[Asynchronous]` and message channels as with any other transport.

**You'll know you need this when:**

* You're handling high-throughput event streaming (10k+ events/sec)
* Multiple services need to consume the same event stream independently
* You have existing Kafka infrastructure and want to use it with Ecotone's attribute-driven model
* RabbitMQ throughput has become a bottleneck for your event volume

{% hint style="success" %}
This module is available as part of **Ecotone Enterprise.**
{% endhint %}

## Materials

### Demo implementation

* [Message Broker demo — RabbitMQ, Kafka, SQS, Redis](https://github.com/ecotoneframework/quickstart-examples/tree/main/MessageBroker)

### Links

* [Message Brokers in PHP: From Hundreds of Lines to Just a Few](https://blog.ecotone.tech/message-brokers-in-php-few-lines-integration/) \[Article]
* [Ecotone Enterprise and Kafka support](https://blog.ecotone.tech/ecotone-enterprise-kafka-distributed-bus-dynamic-channels-and-more-2/) \[Article]
