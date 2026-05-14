---
description: "Core messaging concepts: Messages, Channels, Endpoints, and Gateways"
---

# Messaging concepts

Ecotone is built on a small set of messaging primitives drawn from [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/). Higher-level features like Command/Event/Query buses, Aggregates, Sagas, and Projections are all composed from these building blocks.

If you have used Symfony Messenger or Laravel Queues, the mapping is roughly: a **Message** is the envelope you dispatch, a **Channel** is the transport (queue, in-memory, broker), an **Endpoint/Handler** is the code that processes the message, and a **Gateway** is the typed interface you call to send it.

{% content-ref url="message.md" %}
[message.md](message.md)
{% endcontent-ref %}

{% content-ref url="message-channel.md" %}
[message-channel.md](message-channel.md)
{% endcontent-ref %}

{% content-ref url="message-endpoint/" %}
[message-endpoint](message-endpoint/)
{% endcontent-ref %}

{% content-ref url="consumer.md" %}
[consumer.md](consumer.md)
{% endcontent-ref %}

{% content-ref url="messaging-gateway.md" %}
[messaging-gateway.md](messaging-gateway.md)
{% endcontent-ref %}

{% content-ref url="inbound-outbound-channel-adapter.md" %}
[inbound-outbound-channel-adapter.md](inbound-outbound-channel-adapter.md)
{% endcontent-ref %}
