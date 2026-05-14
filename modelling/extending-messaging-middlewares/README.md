---
description: Extending messaging with Interceptors and Middlewares in Ecotone PHP
---

# Extending Messaging (Middlewares)

Ecotone exposes hooks at every step of a message's lifecycle. Use **Interceptors** (Before/Around/After/Presend) to add cross-cutting behaviour like logging, authorization, retries, or transactions without touching your handlers. Use **Message Headers** to read and write metadata that flows alongside the payload. Use **Custom Gateways** to attach policies (retries, dedup, error channels) to typed bus interfaces.

{% content-ref url="message-headers.md" %}
[message-headers.md](message-headers.md)
{% endcontent-ref %}

{% content-ref url="interceptors/" %}
[interceptors](interceptors/)
{% endcontent-ref %}

{% content-ref url="intercepting-asynchronous-endpoints.md" %}
[intercepting-asynchronous-endpoints.md](intercepting-asynchronous-endpoints.md)
{% endcontent-ref %}

{% content-ref url="extending-message-buses-gateways.md" %}
[extending-message-buses-gateways.md](extending-message-buses-gateways.md)
{% endcontent-ref %}
