# Inbound/Outbound Channel Adapter

When you will go through the code base you will meet `Channel Adapters`.\
Channel adapter are responsible for connecting external message providers to internal messaging system. \
This may be for example `AmqpInboundChannelAdapter`, which connects to RabbitMQ Message Broker to receive and send message internally.&#x20;

## Inbound Channel Adapter

Inbound Channel Adapter is responsible for incoming traffic from external providers. \
It's role is to connect to external provider or to expose API that will be called by external provider. Whatever role is selected, internally it convert incoming message to Ecotone's message and calls message channel (can be done via [Gateway](messaging-gateway.md) or directly [Message Channel](message-channel.md)).

## Outbound Channel Adapter

Outbound Channel Adapter is responsible for outgoing traffic to external providers.

After message flow is done, we may send Message to external provider, this may be for example `AmqpOutboundChannelAdapter` which connects to RabbitMQ Message Broker and converts Ecotone's Message to RabbitMQ message and sends it to the broker.\
Outboud Channel Adapter is implemented by [Message Handler](message-endpoint/).
