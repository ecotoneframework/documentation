---
description: Integrating with other Languages and Frameworks
---

# Non-Ecotone Application integration

When we can install Ecotone in all of our Services (Applications), it's enough to share **Service Map** between them to start communicating, therefore integration is really quick and smooth. \
However there may be cases where installing Ecotone in all Services will not possible, for example we do have some legacy Service working on PHP version lower than 8, or we use different programming languages within our Stack.

## Consuming Messages sent from Ecotone Distributed Bus

On the consumption side of non-Ecotone application, we need to understand the context of the Message. So what type of Message we are dealing with, what should actually be executed, what is the content type of Message's payload.

So with each Distributed Message, Ecotone delivers set of headers, which can be used for answering above questions:\
\
**ecotone.distributed.sourceServiceName**: Name of the Service sending the Message

**ecotone.distributed.targetServiceName**: Name of target Service to which Message is sent

**ecotone.distributed.payloadType**: Type of Message - command/event/message

**ecotone.distributed.routingKey**: Routing key to the Handler which should executed

**contentType**: - Will hold content type of Message's payload (e.g. application/json)

{% hint style="success" %}
All headers can be found in **Ecotone\Modelling\Api\Distribution\DistributedBusHeader**
{% endhint %}

Therefore using Message headers we can find out, what should we execute and how. \
And in the payload of the Message, we will find serialized Event / Command.&#x20;

## Sending Messages to Ecotone based Distributed Service (Application)

On other side we may want to send Message to be consumed by Ecotone's Distributed Handler. \
This can also be done by sending Message with correct headers, so Ecotone can understand it and execute specific Handler.

So Message Headers that's need to be delivered are:

**ecotone.distributed.payloadType**: Type of Message - command/event/message

**ecotone.distributed.routingKey**: Routing key to the Handler which should executed

**contentType**: - Will hold content type of Message's payload (e.g. application/json)&#x20;

**routingSlip**: This is special header which indicate that we want to execute Distributed Handlers. It's value must be set to "**ecotone.distributed.invoke**".

Payload of the Message will contain serialized Event or Command, in format given in **contentType** header. This is enough for Ecotone to resolve and handle given Message.&#x20;
