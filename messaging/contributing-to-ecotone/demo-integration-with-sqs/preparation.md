# Preparation

Let's integrate real life scenario together and make integration with [Amazon SQS](https://aws.amazon.com/sqs/).\
To provide custom Message Channels, that will be backed by Amazon SQS.\
This will allow us to send Ecotone's Messages over SQS.

{% hint style="success" %}
Even, if you do not want to integrate new Module, it's still worth of going through demo, to see how messaging patterns are used and how to write tests.
{% endhint %}

## Preparation

In the demo we will be using package name `SqsDemo.` In repository you will find `_PackageTemplate` that can be used for bootraping your module.\
Start by replacing `_PackageTemplate` with `SqsDemo`.

{% hint style="info" %}
Before starting check [previous chapter](../registering-new-module-package.md).
{% endhint %}

Our composer `autoload` will be like this:

```json
"autoload": {
    "psr-4": {
        "Ecotone\\SqsDemo\\": "src"
    }
},
"autoload-dev": {
    "psr-4": {
        "Test\\Ecotone\\SqsDemo\\": [
            "tests"
        ]
    }
}
```

As Ecotone encourage writing tests on real integrations instead of mocked libraries, we will add docker container, that will provide us with self-hosted `Amazon SQS`.\
We will be using [localstack](https://github.com/localstack/localstack) for hosting local SQS, let's start by adding it to `docker-compose.yaml`.

```
localstack:
  image: localstack/localstack:0.8.10
  networks:
    - default
  environment:
    HOSTNAME_EXTERNAL: 'localstack'
    SERVICES: 'sqs'
```

then by running `docker-compose up -d` it will be now available for us in testing under http://`localstack-sqs-demo:4576` hostname.

## Adding required composer package

Ecotone provides basic integration with [enqueue libraries](https://github.com/php-enqueue) that we can build on top of.&#x20;

{% hint style="info" %}
You may check on [enqueue github](https://github.com/php-enqueue), if it provides integration with service that you would like integrate Ecotone with. If so, then we will need to write less code, as part of the integration it already covered.
{% endhint %}

So the package we are interested is `enqueue/sqs`, let's add this and `ecotone/enqueue`.

```bash
composer require enqueue/sqs && composer require ecotone/enqueue
```

You may take a look on the [API of SQS](https://php-enqueue.github.io/transport/sqs/) over enqueue before we will go to the next step.

{% hint style="success" %}
Always prefer to use well known packages for low level integration with external services. This decrease amount of integration that needs to be maintained and does not reinvent the wheel.
{% endhint %}

## Register Module Package

Start by adding our `sqsDemo` package `ModulePackageList` and `ModuleClassList` in the same way as other modules are added there. This will allow us to run this package from our tests.
