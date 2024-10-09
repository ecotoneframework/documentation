# Configuration

## Installation

```
composer require ecotone/open-telemetry
```

## Configuration with Jaeger

We may use Jaeger for as our 3rd party Tracing System, which will is free open source option.

```
docker run -p 16686:16686 -p 4317:4317 -e COLLECTOR_OTLP_ENABLED=true jaegertracing/all-in-one:latest
```

### Install GRPC and Protobuf

There are two ways of sending traces, over HTTP or GRPC with Protobuf. \
It's recommended to use the second approach, as it avoids putting pressure on your Application performance.&#x20;

```bash
# Install PHP Extensions
install-php-extensions grpc protobuf
# Install PHP Packages in your application
composer require open-telemetry/transport-grpc
composer require open-telemetry/exporter-otlp
```

### Add Tracer Provider to Dependency Container

In order to use **OpenTelemetry Support** we need to add **TracerProviderInterface** to our Dependency Container.&#x20;

{% tabs %}
{% tab title="Symfony" %}
```php
# config/services.yaml
OpenTelemetry\API\Trace\TracerProviderInterface:
    class: OpenTelemetry\API\Trace\TracerProviderInterface
    factory: ['Ecotone\OpenTelemetry\Support\OTelTracer', 'create']
```
{% endtab %}

{% tab title="Laravel" %}
```php
use OpenTelemetry\API\Trace\TracerProviderInterface;
use Ecotone\OpenTelemetry\Support\OTelTracer;

public function register()
{
     $this->app->singleton(TracerProviderInterface::class, function () {
         return OTelTracer::create();
     });
}
```
{% endtab %}

{% tab title="Lite" %}
```php
use OpenTelemetry\API\Trace\TracerProviderInterface;
use Ecotone\OpenTelemetry\Support\OTelTracer;

$application = EcotoneLiteApplication::boostrap(
    [
        TracerProviderInterface::class => OTelTracer::create()
    ]
);
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
By default OTelTracer will use `localhost:4317` to push the logs, this is the port we've exposed in previous step, as this is where Jaeger exposes [Collector](https://opentelemetry.io/docs/collector/).
{% endhint %}

You may now test integration by checking your Jaeger UI at `localhost:16686`.

## Auto-configuration

OpenTelemetry comes with auto configuration which allows us to to hook in into any method and class without modifying existing code. There are multiple preconfigured packages available, that can hook into PDO, Curl, Laravel or Symfony Requests etc.&#x20;

* [**Auto-packages**](https://packagist.org/search/?query=open-telemetry%20auto\&tags=instrumentation) **- Read more about available packages**

To make use of Auto-Instrumentation, we need to provide environment variables, that will be used to automatically configure it. This is needed, because auto-instrumentation may hook even before our Application is booted to provide full insightful details:

```bash
OTEL_SERVICE_NAME: 'customer_service'
OTEL_PHP_AUTOLOAD_ENABLED: true
OTEL_TRACES_EXPORTER: otlp
OTEL_METRICS_EXPORTER: otlp
OTEL_LOGS_EXPORTER: otlp
OTEL_EXPORTER_OTLP_PROTOCOL: grpc
OTEL_EXPORTER_OTLP_ENDPOINT: http://localhost:4317
OTEL_PROPAGATORS: baggage,tracecontext
```

* [**Installation and details**](https://opentelemetry.io/docs/instrumentation/php/automatic/) **- You may read more about installation here**

When using environment variables to configure tracing instead of `OTelTracer` in your Dependency Container, you may use inbuilt Tracer Provider that will create the Tracer based on environments:

{% tabs %}
{% tab title="Symfony" %}
```php
# config/services.yaml
OpenTelemetry\API\Trace\TracerProviderInterface:
    class: OpenTelemetry\API\Globals
    factory: [OpenTelemetry\API\Globals, tracerProvider]
```
{% endtab %}

{% tab title="Laravel" %}
```php
use OpenTelemetry\API\Trace\TracerProviderInterface;
use OpenTelemetry\API\Globals;

public function register()
{
     $this->app->singleton(TracerProviderInterface::class, function () {
         return Globals::tracerProvider();
     });
}
```
{% endtab %}

{% tab title="Lite" %}
```php
use OpenTelemetry\API\Trace\TracerProviderInterface;
use OpenTelemetry\API\Globals;

$application = EcotoneLiteApplication::boostrap(
    [
        TracerProviderInterface::class => Globals::tracerProvider()
    ]
);
```
{% endtab %}
{% endtabs %}

## Flushing Traces

By default Ecotone will `flush traces` after Command/Query buses are executed or after handling Message asynchronously. This way traces will be propagated without any extra configuration. \
\
However we may want to start tracing before Application is bootstrapped. This way we will be able to see time how much time Application have spent on bootstrapping vs actual processing. \
So if you want to take over the process or use of auto-configuration for this, then you can disable force flushing in Ecotone:

```php
#[ServiceContext]
public function traceConfiguration()
{
    return TracingConfiguration::createWithDefaults()
              ->withForceFlushOnBusExecution(false)
              ->withForceFlushOnAsynchronousMessageHandled(false);
}
```

{% hint style="success" %}
If you've enabled auto-instrumentation for Laravel or Symfony, you may disable flush on Bus `withForceFlushOnBusExecution(false)`, as traces will be pushed at the end of the Request.
{% endhint %}
