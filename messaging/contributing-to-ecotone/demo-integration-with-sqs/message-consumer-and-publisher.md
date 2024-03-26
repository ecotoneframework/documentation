# Message Consumer and Publisher

## Message Consumer

[Message Consumer](../../../modelling/microservices-php/message-consumer.md) is implementation that allow to fetch messages by simply annotating method with attribute.

```php
namespace Test\Ecotone\SqsDemo\Fixture\SqsConsumer;

final class SqsConsumerExample
{
    /** @var string[] */
    private array $messagePayloads = [];

    // 1. Message Consumer
    #[MessageConsumer('sqs_consumer')]
    public function collect(string $payload): void
    {
        $this->messagePayloads[] = $payload;
    }

    /**
     * @return string[]
     */
    #[QueryHandler('consumer.getMessagePayloads')]
    public function getMessagePayloads(): array
    {
        return $this->messagePayloads;
    }
}
```

1. `Message Consumer` - By annotation with Attribute we tell Ecotone to activate this method to be used as Message Consumer. `sqs_consumer` will be the endpoint it, so we will be running our consumer by this name.

As Message Consumer can be connected to any external broker, we also need to know that it's supposed to be used with SQS. \
To do this, we will introduce [Extension Object](../../service-application-configuration.md), that will tell us how to configure this `Message Consumer`.

```php
namespace Ecotone\SqsDemo\Configuration;

final class SqsMessageConsumerConfiguration extends EnqueueMessageConsumerConfiguration
{
    private bool $declareOnStartup = SqsInboundChannelAdapterBuilder::DECLARE_ON_STARTUP_DEFAULT;

    public static function create(string $endpointId, string $queueName, string $amqpConnectionReferenceName = SqsConnectionFactory::class): self
    {
        return new self($endpointId, $queueName, $amqpConnectionReferenceName);
    }

    public function withDeclareOnStartup(bool $declareOnStartup): self
    {
        $this->declareOnStartup = $declareOnStartup;

        return $this;
    }

    public function isDeclaredOnStartup(): bool
    {
        return $this->declareOnStartup;
    }
}
```

And now we can create Module that will make use of this configuration in order to register SQS Inbound Channel Adapter.

```php
namespace Ecotone\SqsDemo\Configuration;

// 1. Module Annotation
#[ModuleAnnotation]
final class SqsMessageConsumerModule extends NoExternalConfigurationModule implements AnnotationModule
{
    // 2. Creating Module
    public static function create(AnnotationFinder $annotationRegistrationService, InterfaceToCallRegistry $interfaceToCallRegistry): static
    {
        return new self();
    }

    // 3. Preparing Module
    public function prepare(Configuration $configuration, array $extensionObjects, ModuleReferenceSearchService $moduleReferenceSearchService, InterfaceToCallRegistry $interfaceToCallRegistry): void
    {
        /** @var SqsMessageConsumerConfiguration $extensionObject */
        foreach ($extensionObjects as $extensionObject) {
            $configuration->registerConsumer(
                SqsInboundChannelAdapterBuilder::createWith(
                    $extensionObject->getEndpointId(),
                    $extensionObject->getQueueName(),
                    $extensionObject->getEndpointId(),
                    $extensionObject->getConnectionReferenceName()
                )
                    ->withDeclareOnStartup($extensionObject->isDeclaredOnStartup())
                    ->withHeaderMapper($extensionObject->getHeaderMapper())
                    ->withReceiveTimeout($extensionObject->getReceiveTimeoutInMilliseconds())
            );
        }
    }

    // 4. Choosing Extension Objects
    public function canHandle($extensionObject): bool
    {
        return $extensionObject instanceof SqsMessageConsumerConfiguration;
    }

    public function getModulePackageName(): string
    {
        return ModulePackageList::SQS_PACKAGE;
    }
}
```

1. `Module Annotation` - This tell Ecotone to use this class as Module
2. `Creating Module` - In here we may look for all classes containing given attribute, if there is a need.
3. `Preparing Module` - In here we can adjust configuration using our Module. In our scenario we are using `Message Consumer Configuration` to register `SQS Inbound Channel Adapter` in `Messaging Configuration`.
4. We tell Ecotone what types of `Extension Objects` this Module is looking for.

{% hint style="info" %}
You want to be sure that your `ModulePackageList::SQS_PACKAGE` is available in `ModulePackageList::allPackages`, and `ModulePackageList::getModuleClassesForPackage, so it can be correctly resolved.`
{% endhint %}

Before we will test this, we need a way to send Message to SQS Queue first. In order to do so, we need to set up `Message Publisher`.

{% hint style="success" %}
We have register Inbound Channel Adapter, however we have not connected it anyhow `SqsConsumerExample`. \
This is the same for all types of Message Consumers, so there is a separate module for that `MessageConsumerModule`. \
This module connect this class to channel named by `endpoint id`. \
Our inbound Channel Adapter (3rd parameter), we are using endpoint id, as request channel after fetching message from the queue. Thanks to that it's all connected.
{% endhint %}

## Message Publisher

[Message Publisher](../../../modelling/microservices-php/message-publisher.md) is Gateway implementation that allow us to create abstraction that will send messages to SQS Queue via simple interface.

First let's create `Sqs Publisher Configuration`:

```php
namespace Ecotone\SqsDemo\Configuration;

final class SqsMessagePublisherConfiguration
{
    private bool $autoDeclareOnSend = true;
    private string $headerMapper = '';

    private function __construct(private string $connectionReference, private string $queueName, private ?string $outputDefaultConversionMediaType, private string $referenceName)
    {
    }

    public static function create(string $publisherReferenceName = MessagePublisher::class, string $queueName = '', ?string $outputDefaultConversionMediaType = null, string $connectionReference = SqsConnectionFactory::class): self
    {
        return new self($connectionReference, $queueName, $outputDefaultConversionMediaType, $publisherReferenceName);
    }

    public function getConnectionReference(): string
    {
        return $this->connectionReference;
    }

    public function withAutoDeclareQueueOnSend(bool $autoDeclareQueueOnSend): self
    {
        $this->autoDeclareOnSend = $autoDeclareQueueOnSend;

        return $this;
    }

    /**
     * @param string $headerMapper comma separated list of headers to be mapped.
     *                             (e.g. "\*" or "thing1*, thing2" or "*thing1")
     */
    public function withHeaderMapper(string $headerMapper): self
    {
        $this->headerMapper = $headerMapper;

        return $this;
    }

    public function isAutoDeclareOnSend(): bool
    {
        return $this->autoDeclareOnSend;
    }

    public function getHeaderMapper(): string
    {
        return $this->headerMapper;
    }

    public function getOutputDefaultConversionMediaType(): ?string
    {
        return $this->outputDefaultConversionMediaType;
    }

    public function getQueueName(): string
    {
        return $this->queueName;
    }

    public function getReferenceName(): string
    {
        return $this->referenceName;
    }
}
```

and then we can create Module, that will use of this config in order to register our `Sqs Outbound Channel Adapter`.

```php
namespace Ecotone\SqsDemo\Configuration;

#[ModuleAnnotation]
final class SqsMessagePublisherModule extends NoExternalConfigurationModule implements AnnotationModule
{
    public static function create(AnnotationFinder $annotationRegistrationService, InterfaceToCallRegistry $interfaceToCallRegistry): static
    {
        return new self();
    }

    public function prepare(Configuration $configuration, array $extensionObjects, ModuleReferenceSearchService $moduleReferenceSearchService, InterfaceToCallRegistry $interfaceToCallRegistry): void
    {
        // 1. Extension Object Resolver
        $serviceConfiguration = ExtensionObjectResolver::resolveUnique(ServiceConfiguration::class, $extensionObjects, ServiceConfiguration::createWithDefaults());

        /** @var SqsMessagePublisherConfiguration $messagePublisher */
        foreach (ExtensionObjectResolver::resolve(SqsMessagePublisherConfiguration::class, $extensionObjects) as $messagePublisher) {
            $mediaType = $messagePublisher->getOutputDefaultConversionMediaType() ?: $serviceConfiguration->getDefaultSerializationMediaType();

            // 2. Registering Messaging Gateways
            $configuration
                ->registerGatewayBuilder(
                    GatewayProxyBuilder::create($messagePublisher->getReferenceName(), MessagePublisher::class, 'send', $messagePublisher->getReferenceName())
                        ->withParameterConverters([
                            GatewayPayloadBuilder::create('data'),
                            GatewayHeaderBuilder::create('sourceMediaType', MessageHeaders::CONTENT_TYPE),
                        ])
                )
                ->registerGatewayBuilder(
                    GatewayProxyBuilder::create($messagePublisher->getReferenceName(), MessagePublisher::class, 'sendWithMetadata', $messagePublisher->getReferenceName())
                        ->withParameterConverters([
                            GatewayPayloadBuilder::create('data'),
                            GatewayHeadersBuilder::create('metadata'),
                            GatewayHeaderBuilder::create('sourceMediaType', MessageHeaders::CONTENT_TYPE),
                        ])
                )
                ->registerGatewayBuilder(
                    GatewayProxyBuilder::create($messagePublisher->getReferenceName(), MessagePublisher::class, 'convertAndSend', $messagePublisher->getReferenceName())
                        ->withParameterConverters([
                            GatewayPayloadBuilder::create('data'),
                            GatewayHeaderValueBuilder::create(MessageHeaders::CONTENT_TYPE, MediaType::APPLICATION_X_PHP),
                        ])
                )
                ->registerGatewayBuilder(
                    GatewayProxyBuilder::create($messagePublisher->getReferenceName(), MessagePublisher::class, 'convertAndSendWithMetadata', $messagePublisher->getReferenceName())
                        ->withParameterConverters([
                            GatewayPayloadBuilder::create('data'),
                            GatewayHeadersBuilder::create('metadata'),
                            GatewayHeaderValueBuilder::create(MessageHeaders::CONTENT_TYPE, MediaType::APPLICATION_X_PHP),
                        ])
                )
                // 3. Registering Outbound Channel Adapter
                ->registerMessageHandler(
                    SqsOutboundChannelAdapterBuilder::create($messagePublisher->getQueueName(), $messagePublisher->getConnectionReference())
                        ->withEndpointId($messagePublisher->getReferenceName() . '.handler')
                        ->withInputChannelName($messagePublisher->getReferenceName())
                        ->withAutoDeclareOnSend($messagePublisher->isAutoDeclareOnSend())
                        ->withHeaderMapper($messagePublisher->getHeaderMapper())
                        ->withDefaultConversionMediaType($mediaType)
                );
        }
    }

    public function canHandle($extensionObject): bool
    {
        return
            $extensionObject instanceof SqsMessagePublisherConfiguration
            || $extensionObject instanceof ServiceConfiguration;
    }

    public function getModulePackageName(): string
    {
        return ModulePackageList::SQS_PACKAGE;
    }
}
```

1. `Extension Object Resolver` - This is a helper class that allows use to fetch from extension object, concrete class or classes of given type.
2. `Registering Messaging Gateways` - [Messaging Gateways](../../messaging-concepts/messaging-gateway.md) are entrypoints to messaging system as `Message Publisher` is an interface that will be called by end-user, this means it's a Messaging Gateway. We register all method that this interface has together with parameter converters
3. `Registering Outbound Channel Adapter` - Using our `Message Publisher Configuration` we register our `Sqs Outbound Channel Adapter`.\
   The input channel name is the same as request channel name of Gateway. This way, when interface's method will be called under the hood we will call our Outbound Channel Adapter.

## Testing Message Publisher and Consumer

Our test case can look like this:

```php
public function testing_sending_message_using_publisher_and_receiving_using_consumer()
{
    $endpointId = 'sqs_consumer';
    $queueName = Uuid::uuid4()->toString();
    $ecotoneLite = EcotoneLite::bootstrapForTesting(
        [SqsConsumerExample::class],
        [
            new SqsConsumerExample(),
            SqsConnectionFactory::class => $this->getConnectionFactory(),
        ],
        ServiceConfiguration::createWithDefaults()
            ->withSkippedModulePackageNames(ModulePackageList::allPackagesExcept([ModulePackageList::SQS_PACKAGE]))
            ->withExtensionObjects([
                SqsMessageConsumerConfiguration::create($endpointId, $queueName),
                SqsMessagePublisherConfiguration::create(queueName: $queueName),
            ])
    );

    $payload = 'random_payload';
    $messagePublisher = $ecotoneLite->getMessagePublisher();
    $messagePublisher->send($payload);

    $ecotoneLite->run($endpointId, ExecutionPollingMetadata::createWithDefaults()->withHandledMessageLimit(1)->withExecutionTimeLimitInMilliseconds(1));
    $this->assertEquals([$payload], $ecotoneLite->getQueryBus()->sendWithRouting('consumer.getMessagePayloads'));

    $ecotoneLite->run($endpointId, ExecutionPollingMetadata::createWithDefaults()->withHandledMessageLimit(1)->withExecutionTimeLimitInMilliseconds(1));
    $this->assertEquals([$payload], $ecotoneLite->getQueryBus()->sendWithRouting('consumer.getMessagePayloads'));
}
```

We are using `Message Publisher` to send the message and then we are consuming it using our `Message Consumer`.

## Summary

We have introduced `Message Channel` and `Message Consumer` and `Publisher`.\
However the the hood we had chance to get known with `Messaging Gateways`, `Inbound and Outbound Channel Adapters` and `Message Handlers`.\
\
Using few patterns you may actually build really customized Messaging Configurations. This is power of decoupled system, when you get familiar with them, they open much more possibilities.
