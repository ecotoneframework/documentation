# Contributing to Ecotone

## Preparing development environment

Start by cloning [ecotone-dev repository](https://github.com/ecotoneframework/ecotone-dev).&#x20;

1. To run your local environment with tests:
2.  Copy `.env.dist` to `.env` and start docker containers

    ```bash
    docker-compose up -d
    ```
3.  To run tests for `monorepo`:

    ```bash
    docker exec -it ecotone_development composer tests:local
    ```
4.  To run tests for given `module`

    ```bash
    docker exec -it -w=/data/app/packages/Dbal ecotone_development composer tests:ci
    ```
5.  Clear development environment

    ```bash
    docker-compose down
    ```

## Debugging code

Development containers comes with [xdebug](https://xdebug.org/) installed, so you can debug directly from IDE.

*   To have enabled debugging all the time, change line in your `.env` file to&#x20;

    ```bash
    XDEBUG_ENABLED="1"
    ```

    and rebuild containers:

    ```bash
    docker-compose down && docker-compose up -d
    ```
*   As having xdebug enabled all the time, may slow your test execution, you may run it conditionally for given test case

    ```bash
    docker exec -it ecotone_development xdebug vendor/bin/phpunit --filter test_calling_command_on_aggregate_and_receiving_aggregate_instance
    ```

{% hint style="info" %}
If you're asked about mapping path, map your xdebug server to "`/data/app"` and name it "`project`", if you have not changed xdebug project name.
{% endhint %}

## Ecotone Concepts

Even though Ecotone introduces its own concepts, many of them already existed. A good source of information is Gregor Hohpe's [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) website and accompanying book.

{% hint style="success" %}
For example, if you're interested in what Service Activator is, you make [take a look here.](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessagingAdapter.html)
{% endhint %}

You may also take a look on [previous chapters](../overview.md) to get familiar with fundamental building blocks.

{% hint style="success" %}
Due to the fact that Ecotone is based on EIP patterns, you will find a lot of similarities to well known frameworks in other languages, like `C#'s` [NServiceBus](https://docs.particular.net/nservicebus/) or Java's [Spring Integration](https://spring.io/projects/spring-integration) (which is foundation for [Spring Cloud](https://spring.io/projects/spring-cloud)), as they are also built on top of EIP patterns.
{% endhint %}

## Contribution guidelines

* _Make use of real providers in tests_ - This means, if integrating with RabbitMQ/SQS write tests that actually make use of this Message Brokers. This will ensure, that tests will stay the same, even if underlying 3rd party library/SDK will change.
* _Focus on client level code_ - Your tests do not need to test every single class you add to the repository. It's more valuable from perspective of long term maintenance to focus on tests that will be testing on the high level (using [Ecotone Lite](../../modelling/testing-support/testing-messaging.md)). \
  Those tests will be also similar to the ones, you will be writing in your own Ecotone based projects.
* _Ask questions in case you need some help_ - You can use [Ecotone's community channel](https://discord.gg/CctGMcrYnV) to get quick feedback and help with your implementation.

## Articles on how internal works

* [Building Message-Driven Framework - Foundation](https://blog.ecotone.tech/building-message-driven-framework-foundation/)
