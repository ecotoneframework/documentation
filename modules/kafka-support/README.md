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

### Links

* [Ecotone Enterprise and Kafka support](https://blog.ecotone.tech/ecotone-enterprise-kafka-distributed-bus-dynamic-channels-and-more-2/) \[Article]
