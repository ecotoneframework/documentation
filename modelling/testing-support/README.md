---
coverY: -559.3548387096773
---

# Testing Support

Ecotone provides comprehensive testing tools to help you write reliable tests for your message-driven applications. This section covers everything from basic unit testing to complex integration scenarios.

## Getting Started

**New to Ecotone Testing?** Start here:

{% content-ref url="testing-messaging.md" %}
[testing-messaging.md](testing-messaging.md) - **Start Here:** Basic testing setup and concepts
{% endcontent-ref %}

## Core Testing Patterns

Once you understand the basics, explore these testing patterns:

{% content-ref url="testing-aggregates-and-sagas-with-message-flows.md" %}
[testing-aggregates-and-sagas-with-message-flows.md](testing-aggregates-and-sagas-with-message-flows.md) - Testing business logic flows
{% endcontent-ref %}

## Specialized Testing

For specific architectural patterns:

{% content-ref url="testing-event-sourcing-applications.md" %}
[testing-event-sourcing-applications.md](testing-event-sourcing-applications.md) - Event sourcing specific testing
{% endcontent-ref %}

{% content-ref url="testing-asynchronous-messaging.md" %}
[testing-asynchronous-messaging.md](testing-asynchronous-messaging.md) - Async messaging testing strategies
{% endcontent-ref %}

### Performance Tips
- Use in-memory repositories and event stores for fast tests
- Skip unnecessary modules with `withSkippedModulePackageNames()`
- Leverage Ecotone's configuration caching for large test suites

## Materials

## Demo

* [Testing Aggregates, Saga, Projections](https://github.com/ecotoneframework/quickstart-examples/tree/main/Testing)

### Links

* [Testing Asynchronous Message Driven Architecture](https://blog.ecotone.tech/testing-messaging-architecture-in-php/) \[Article]
* [How to make sending Messages to the Message Broker resilient](https://blog.ecotone.tech/my-database-is-not-a-message-broker/) \[Article]
* [What is and how to use Outbox Pattern](https://blog.ecotone.tech/implementing-outbox-pattern-in-php-symfony-laravel-ecotone/) \[Article]
