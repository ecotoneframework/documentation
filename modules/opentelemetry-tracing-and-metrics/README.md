# OpenTelemetry (Tracing and Metrics)

<figure><img src="../../.gitbook/assets/Screenshot from 2023-11-04 17-59-51.png" alt=""><figcaption><p>Trace your flows using OpenTelemetry</p></figcaption></figure>

[**OpenTelemetry**](https://opentelemetry.io/docs/instrumentation/php/) is tracing abstraction to collect and aggregate data such as metrics and traces.\
As OpenTelemetry is tracing abstraction it does not force to use specific Tracing 3rd party. Depending on the needs and preferences, we may choose provides like [DataDog](https://www.datadoghq.com/), [Jaeger](https://www.jaegertracing.io/), [Zipkin](https://zipkin.io/) etc.

**Ecotone** brings full tracing using **OpenTelemetry** to your PHP applications. It will provide details about your Message flows no matter if they are synchronous or asynchronous.

## Materials

### Demo implementation

* [Demo implementation with Symfony and Laravel](https://github.com/ecotoneframework/php-ddd-cqrs-event-sourcing-symfony-laravel-ecotone)

### Links

* [How OpenTelemetry can be used with Ecotone](https://blog.ecotone.tech/tracing-using-opentelemetry/)
