---
description: Advanced queuing strategies for multi-tenant message processing
---

# Advanced Queuing Strategies

A free-tier tenant fires 10,000 messages and your queue backlog blocks every other tenant for an hour. Or you onboard a premium tenant and want their messages on a dedicated channel without rewriting handlers. Or you're rolling out a deploy and want to drain one tenant's queue cleanly while others keep flowing.

These scenarios outgrow the static channel model. Ecotone's **Dynamic Message Channels** let you route, throttle, and isolate per tenant at runtime — picking the channel from a header, applying credit-based rate limits, or sending specific tenants to dedicated workers — without changing the handler code.

The full feature is documented under Asynchronous Handling, since the same primitives apply to non-tenant routing as well:

{% content-ref url="../../modelling/asynchronous-handling/dynamic-message-channels.md" %}
[dynamic-message-channels.md](../../modelling/asynchronous-handling/dynamic-message-channels.md)
{% endcontent-ref %}
