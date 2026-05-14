---
description: Distributed Bus for cross-service messaging in PHP
---

# Distributed Bus

## The Problem

Service A publishes `OrderWasPlaced`. Service B needs to react to it. Today you POST it over HTTP — and when B is down, A times out too. You've already written serialization glue, custom retry middleware, and routing tables for three service pairs, and adding a fourth feels like another week of plumbing.

## How Ecotone Solves It

A **Distributed Bus** is a [Message Gateway](../../../messaging/messaging-concepts/messaging-gateway.md) like CommandBus or EventBus, but the destination is *another service* instead of a local handler. You publish or send the same way you do locally; Ecotone routes the message across your existing broker (RabbitMQ, Kafka, SQS, Redis, Symfony Messenger, Laravel Queues) without per-pair configuration.

Read more in the relevant module section.

## Support

To find out more, read section related to specific implementation of Distributed Bus:

* [Distributed Bus with Service Map](distributed-bus-with-service-map/) - Works with (RabbitMQ, Amazon SQS, Redis, Dbal, Kafka, Symfony Message Channels, Laravel Queues)
* [RabbitMQ Distributed Bus](amqp-distributed-bus-rabbitmq/) - Works with RabbitMQ only
