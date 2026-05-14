---
description: Multi-tenancy scenarios and advanced configurations
---

# Different Scenarios

Once tenant resolution is configured, real systems run into a wider set of needs: tenants on different databases, tenants derived from inbound payloads, propagating tenant context across asynchronous events, and isolating dead letters per tenant. The pages below cover each scenario.

{% content-ref url="hooking-into-tenant-switch.md" %}
[hooking-into-tenant-switch.md](hooking-into-tenant-switch.md)
{% endcontent-ref %}

{% content-ref url="shared-and-multi-database-tenants.md" %}
[shared-and-multi-database-tenants.md](shared-and-multi-database-tenants.md)
{% endcontent-ref %}

{% content-ref url="accessing-current-tenant-in-message-handler.md" %}
[accessing-current-tenant-in-message-handler.md](accessing-current-tenant-in-message-handler.md)
{% endcontent-ref %}

{% content-ref url="deriving-tenant-from-inbound-messages.md" %}
[deriving-tenant-from-inbound-messages.md](deriving-tenant-from-inbound-messages.md)
{% endcontent-ref %}

{% content-ref url="events-and-tenant-propagation.md" %}
[events-and-tenant-propagation.md](events-and-tenant-propagation.md)
{% endcontent-ref %}

{% content-ref url="multi-tenant-aware-dead-letter.md" %}
[multi-tenant-aware-dead-letter.md](multi-tenant-aware-dead-letter.md)
{% endcontent-ref %}
