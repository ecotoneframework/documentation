---
description: CQRS DDD PHP
---

# Overview

## DDD & CQRS Concepts

Ecotone enables Domain-Driven Design (DDD) and Command Query Responsibility Segregation in Messaging Architecture. While a full explanation of these concepts is beyond the scope and intent of this reference guide, we do want to provide a summary of the most important concepts in the context of an Ecotone application.

## Strategic concepts <a href="#strategic-concepts" id="strategic-concepts"></a>

Strategic Design is a set of principles for maintaining model integrity, distilling the Domain Model, and working with multiple models. It provides the concepts to design boundaries of components and the interaction between them.&#x20;

### Domains and Subdomains <a href="#domains-and-subdomains" id="domains-and-subdomains"></a>

The formal definition of a Domain in the context of DDD is:

> A sphere of knowledge, influence, or activity. The subject area to which the user applies a program is the domain of the software.

It refers to the specific subject that the project is being developed for.&#x20;

### Model <a href="#model" id="model"></a>

A model is:

> A system of abstractions that describes selected aspects of a domain and can be used to solve problems related to that domain;

In other words, a model captures what's important and helpful to us in solving a specific problem within our domain. This definition in itself suggests that an application should consist of more than one model, as each problem has a different ideal model to address it.

## Tactical concepts <a href="#tactical-concepts" id="tactical-concepts"></a>

To build a model, DDD provide a number of useful building blocks.&#x20;

### Aggregates <a href="#aggregates" id="aggregates"></a>

The formal definition, by DDD is:

> A cluster of associated objects that are treated as a unit for the purpose of data changes. External references are restricted to one member of the Aggregate, designated as the root. A set of consistency rules applies within the Aggregate's boundaries.

An Aggregate is an entity or group of entities that is always kept in a consistent state (within a single ACID transaction). The Aggregate Root is the entry point, which exposes public set of actions behaviours possible within specific Aggregate. This makes the aggregate the prime building block for implementing a command model in any CQRS based application.

### Sagas

While ACID transactions are not necessary or even impossible in some cases, some form of transaction management is still required. Typically, these transactions are referred to as BASE transactions: Basically Available, Soft state, Eventual consistency. Contrary to ACID, BASE transactions cannot be easily rolled back.

In CQRS, Sagas can be used to manage these BASE transactions. They respond to events and may dispatch commands, invoke external applications, etc. \
It is common for Sagas to be used as coordination mechanism between different aggregates in order to eventually achieve consistency.

## Materials

### Links

* [Message Processing in PHP - Symfony Messenger, Laravel Queues and Ecotone comparison](https://blog.ecotone.tech/message-processing-in-php-symfony-laravel-ecotone/)

