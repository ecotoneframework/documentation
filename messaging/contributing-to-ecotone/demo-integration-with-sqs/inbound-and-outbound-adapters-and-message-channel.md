# Inbound and Outbound Adapters and Message Channel

[Inbound Channel Adapters ](../../messaging-concepts/inbound-outbound-channel-adapter.md#inbound-channel-adapter)are entrypoint for Messaging System. They do connect to external sources and consume Messages from them, which then are converted to Ecotone's Messages.

On other hand, [Outbound Channel Adapters](../../messaging-concepts/inbound-outbound-channel-adapter.md#outbound-channel-adapter) are sending Ecotone's Messages to external sources, by converting to Messages specific to given integration.

### Inbound Channel Adapter

```php
namespace Ecotone\SqsDemo;

// 1. Extending EnqueueInboundChannelAdapter
final class SqsInboundChannelAdapter extends EnqueueInboundChannelAdapter
{
    // 2. Initialization method
    public function initialize(): void
    {
        /** @var SqsContext $context */
        $context = $this->connectionFactory->createContext();

        $context->declareQueue($context->createQueue($this->queueName));
    }
}
```

1. `Extending EnqueueInboundChannelAdapter` - As Enqueue comes great abstraction over queues, Ecotone comes with sensible defaults on top of that. You may take a look inside, if you want to get more familiar with the details.
2. `Initialization method` - During initialization we want to setup the Queue and we do it using enqueue abstraction.

{% hint style="success" %}
Ecotone always tries automatic setup. You may retake control when you need, but when you don't it will setup everything for you, so you can focus on the business code instead.
{% endhint %}

### Inbound Channel Adapter Builder

We need to create a Builder method for that too:

```php
namespace Ecotone\SqsDemo;

// 1. Extending EnqueueInboundChannelAdapterBuilder
final class SqsInboundChannelAdapterBuilder extends EnqueueInboundChannelAdapterBuilder
{
    // 2. Factory method
    public static function createWith(string $endpointId, string $queueName, ?string $requestChannelName, string $connectionReferenceName = SqsConnectionFactory::class): self
    {
        return new self($queueName, $endpointId, $requestChannelName, $connectionReferenceName);
    }

    // 3. Building SqsInboundChannelAdapter    
    public function compile(MessagingContainerBuilder $builder): Definition
    {
        // 4. connection factory
        $connectionFactory = new Definition(CachedConnectionFactory::class, [
            new Definition(HttpReconnectableConnectionFactory::class, [
                new Reference($this->connectionReferenceName),
            ]),
        ], 'createFor');
        // 5. Inbound Message Converter
        $inboundMessageConverter = new Definition(InboundMessageConverter::class, [
            $this->endpointId,
            $this->acknowledgeMode,
            // 6. Header Mapper
            DefaultHeaderMapper::createWith($this->headerMapper, []),
            EnqueueHeader::HEADER_ACKNOWLEDGE,
            Reference::to(LoggingGateway::class),
        ]);

        return new Definition(SqsInboundChannelAdapter::class, [
            $connectionFactory,
            $this->declareOnStartup,
            $this->messageChannelName,
            $this->receiveTimeoutInMilliseconds,
            $inboundMessageConverter,
            new Reference(ConversionService::REFERENCE_NAME),
        ]);
    }
}
```

1. `Extending EnqueueInboundChannelAdapterBuilder` - This just like in previous example, provide sensible defaults, so we don't need to reimplement the wheel.
2. Factory method provide provides&#x20;

* `endpointId` - Which is the name for the consumer, you will be running it using this name
* `queueName` - The name of the queue created in SQS
* `requestChannelName` - This is the message channel that message will be send to. For now we don't need to bother about this
* `connectionReferenceName` - This provide the default name, under which the connection factory will be registered in Dependency Container. This is how we will be looking it up.

3\. `Building channel adapter` - This method will build our Channel Adapter, using available configuration.\
The compile part is way of defining an DI Service within Ecotone.  We tell Ecotone how given object should be constructed and run time time this configuration will be used to construct the object.

4\. `Connection Factory` - Connection factory is retrieved from DI Container based on `connectionReferenceName`.

5\. `Inbound Message Converter` - It converts incoming message to `Ecotone's Message`, so we can make use of it for our Messaging communication.

6\. `Header Mapper` - Tells how we should map headers in our incoming message. Can be customized, if needed.

{% hint style="success" %}
Ecotone make use of Builder patterns for the configurations. \
Builder patterns are POPO objects (not [resources](https://www.php.net/manual/en/language.types.resource.php)), thanks to that Ecotone can cache whole configuration in [preparation phase](../ecotone-phases.md#preparation-phase).
{% endhint %}

### Outbound Channel Adapter

We've created Inbound, however consumer without any message to consume will have not much work to do. That's why we need to create Outbound Channel Adapter that will send Messages to our `SQS Queue`.

```php
namespace Ecotone\SqsDemo;

// 1. Sqs Outbound Channel Adapter
final class SqsOutboundChannelAdapter extends EnqueueOutboundChannelAdapter
{
    public function __construct(CachedConnectionFactory $connectionFactory, private string $queueName, bool $autoDeclare, OutboundMessageConverter $outboundMessageConverter)
    {
        parent::__construct(
            $connectionFactory,
            // 2. Sqs Destination
            new SqsDestination($queueName),
            $autoDeclare,
            $outboundMessageConverter
        );
    }

    // 3. Initialization
    public function initialize(): void
    {
        /** @var SqsContext $context */
        $context = $this->connectionFactory->createContext();

        $context->declareQueue($context->createQueue($this->queueName));
    }
}
```

1. `Sqs Outbound Channel Adapter` - We use of abstract Enqueue Outbound Channel Adapter, that will provide us with sensible defaults
2. `Constructor` - The only difference to previous example is `Destination.` This instructs where the message should be sent using Enqueue abstraction.
3. `Initialization` - When sending message Ecotone can initialize the queue for you too, this ensures that message will be delivered, even when customer is not running yet. This configuration can be of course turned off.

### Outbound Channel Adapter Builder

```php
namespace Ecotone\SqsDemo;

final class SqsOutboundChannelAdapterBuilder extends EnqueueOutboundChannelAdapterBuilder
{
    private function __construct(private string $queueName, private string $connectionFactoryReferenceName)
    {
        $this->initialize($connectionFactoryReferenceName);
    }

    public static function create(string $queueName, string $connectionFactoryReferenceName = SqsConnectionFactory::class): self
    {
        return new self($queueName, $connectionFactoryReferenceName);
    }
    public function compile(MessagingContainerBuilder $builder): Definition
    {
        $connectionFactory = new Definition(CachedConnectionFactory::class, [
            new Definition(HttpReconnectableConnectionFactory::class, [
                new Reference($this->connectionFactoryReferenceName),
            ]),
        ], 'createFor');

        $outboundMessageConverter = new Definition(OutboundMessageConverter::class, [
            $this->headerMapper,
            $this->defaultConversionMediaType,
            $this->defaultDeliveryDelay,
            $this->defaultTimeToLive,
            $this->defaultPriority,
            [],
        ]);

        return new Definition(SqsOutboundChannelAdapter::class, [
            $connectionFactory,
            $this->queueName,
            $this->autoDeclare,
            $outboundMessageConverter,
            new Reference(ConversionService::REFERENCE_NAME),
        ]);
    }
}
```

There is nothing new, as it's just other side of the coin.&#x20;

## Message Channel

So message channel is abstraction that connect both inbound and outbound. We use this abstraction when given message is published and consumed within same application.&#x20;

Let's prepare Builder for our SQS Message Channel

```php
namespace Ecotone\SqsDemo;

// 1. Sqs Backed Message Channel Builder
final class SqsBackedMessageChannelBuilder extends EnqueueMessageChannelBuilder
{
    // 2. Constructor
    private function __construct(string $channelName, string $connectionReferenceName)
    {
        parent::__construct(
            SqsInboundChannelAdapterBuilder::createWith(
                $channelName,
                $channelName,
                null,
                $connectionReferenceName
            ),
            SqsOutboundChannelAdapterBuilder::create(
                $channelName,
                $connectionReferenceName
            )
        );
    }

    // 3. Factory method
    public static function create(string $channelName, string $connectionReferenceName = SqsConnectionFactory::class): self
    {
        return new self($channelName, $connectionReferenceName);
    }
}
```

1. `Sqs Backed Message Channel Builder` - We extend with Enqueue Message Channel to provide sensible defaults
2. `Constructor` - This makes use of our Inbound and Outbound Channel Adapters
3. `Factory method` - We provide factory method with default for our connection name

{% hint style="success" %}
Message Channel joins both ends. We use for handlers inside the application, as send and receive message within same application.\
That's the difference between Message Publisher and Consumer, as Message Publisher can be used for sending to queue that we don't consume from and Message Consumer can be used to fetch from queue that we don't send too.&#x20;
{% endhint %}

## Running tests for our Message Channel

Before we will add any tests, let's provide a bootstrap file, that will provide us with `ConnectionFactory` set up for our local `SQS`.

```php
namespace Test\Ecotone\SqsDemo;

abstract class AbstractConnectionTest extends TestCase
{
    private ?SqsConnectionFactory $connectionFactory = null;

    public function getConnectionFactory(): ConnectionFactory
    {
        if (!$this->connectionFactory) {
            $dsn = getenv('SQS_DSN') ? getenv('SQS_DSN') : 'sqs:?key=key&secret=secret&region=us-east-1&endpoint=http://localstack:4576&version=latest';

            $this->connectionFactory = new SqsConnectionFactory($dsn);
        }

        return $this->connectionFactory;
    }
}
```

`Ecotone` encourage to write high level tests for happy paths (test scenarios that are not verifying exceptional cases), as those tests are much easier to maintainable and can be understand much easier.

```php
namespace Test\Ecotone\SqsDemo\Integration;

// 1. Sqs Backend Message Channel Test
final class SqsBackedMessageChannelTest extends AbstractConnectionTest
{
    public function test_sending_and_receiving_message()
    {
        // 2. Initial data
        $queueName = Uuid::uuid4()->toString();
        $messagePayload = 'some';

        // 3. Ecotone Lite
        $ecotoneLite = EcotoneLite::bootstrapForTesting(
            // 4. Sqs Connection Factory
            containerOrAvailableServices: [
                SqsConnectionFactory::class => $this->getConnectionFactory(),
            ],
            // 5. Skipped Module packages and Extension Objects
            configuration: ServiceConfiguration::createWithDefaults()
                ->withSkippedModulePackageNames(ModulePackageList::allPackagesExcept([ModulePackageList::SQS_PACKAGE]))
                ->withExtensionObjects([
                    SqsBackedMessageChannelBuilder::create($queueName)
                ])
        );

        // 6. Retrieving SQS Message Channel
        /** @var PollableChannel $messageChannel */
        $messageChannel = $ecotoneLite->getMessageChannelByName($queueName);

        // 7. Sending to SQS Message Channel
        $messageChannel->send(MessageBuilder::withPayload($messagePayload)->build());

        // 8. Polling from SQS Message Channel
        $this->assertEquals(
            $messagePayload,
            $messageChannel->receiveWithTimeout(1)->getPayload()
        );
    }
}
```

1. `Sqs Backed Message Channel Test` - We configure our test case.
2. `Initial data` - We set up initial data, we will be using in tests. It's important to generate the queue name, to avoid false-positive tests.
3. `Ecotone Lite` - We bootstrap Ecotone Lite, which can run our Ecotone Application with setup that we will provide.
4. `Sqs Connection Factory` - We provide to Ecotone's Dependency Contaiiner, connection factory that we've set up in previous example.
5. `Module Packages and Extension Objects` - This way we can pass some additional configuration to Ecotone. In this case we are telling Ecotone to load only our `SQS Module` and we are passing `SQS Message Channel Builder`.
6. `Retrieving SQS Channel` - After Ecotone Lite is up, we can fetch built Message Channel to make use of it
7. `Sending to the channel` - By using `MessageBuilder` we can construct Ecotone's Message. Normally we don't deal with Message directly, as Ecotone abstract this away.
8. `Polling from the channel` - And now we can pull from the channel, to see if the message it there

{% hint style="success" %}
We were using Message Channel directly, which in most of the cases will not really happen. \
In typical usage, there will be automatically registered consumer that will be consuming from this channel.
{% endhint %}

## Message Channel with Consumer

Let's add one more test case for running Ecotone on high level, like we would do with in typical application making Event or Command Handler asynchronous using our new channel.\
\
Let's start by adding `Asynchronous Command Handler` - `OrderService.`

<pre class="language-php"><code class="lang-php">namespace Test\Ecotone\SqsDemo\Fixture\Order;
<strong>
</strong><strong>// 1. Asynchronous channel
</strong><strong>#[Asynchronous('orders')]
</strong>class OrderService
{
    /**
     * @var string[]
     */
    private $orders = [];

    // 2. Our asynchronous command handler
    #[CommandHandler('order.register', 'orderReceiver')]
    public function register(string $placeOrder): void
    {
        $this->orders[] = $placeOrder;
    }

    // 3. Query Handler for assertions
    #[QueryHandler('order.getOrders')]
    public function getRegisteredOrders(): array
    {
        return $this->orders;
    }
}
</code></pre>

1. `Asynchronous Channel` - We define that all Command Handlers in this class will be using asynchronous channel called `orders`
2. `Asynchronous Command Handler` - We define Command Handler that we will be calling using `Command Bus`
3. `Query Handler` - We will be calling this query handler to verify the result

```php
public function test_sending_and_receiving_message_from_using_asynchronous_command_handler()
{
    $queueName = 'orders';

    $ecotoneLite = EcotoneLite::bootstrapForTesting(
        // 1. Resolver
        [OrderService::class],
        [
            // 2. Dependency Container
            new OrderService(),
            AmqpConnectionFactory::class => $this->getRabbitConnectionFactory(),
        ],
        ServiceConfiguration::createWithDefaults()
            ->withSkippedModulePackageNames(ModulePackageList::allPackagesExcept([
                ModulePackageList::AMQP_PACKAGE, 
                // 3. Asynchronous Package
                ModulePackageList::ASYNCHRONOUS_PACKAGE
            ]))
            ->withExtensionObjects([
                AmqpBackedMessageChannelBuilder::create($queueName),
            ])
    );

    // 3. Clear Queue
    $this->getConnectionFactory()->createContext()->purgeQueue(new SqsDestination($queueName));

    // 5. Send Command
    $ecotoneLite->getCommandBus()->sendWithRouting('order.register', "milk");
    /** Message should be waiting in the queue */
    $this->assertEquals([], $ecotoneLite->getQueryBus()->sendWithRouting('order.getOrders'));

    // 6. Run consumer
    $ecotoneLite->run('orders', ExecutionPollingMetadata::createWithDefaults()->withTestingSetup());
    /** Message should cosumed from the queue */
    $this->assertEquals(['milk'], $ecotoneLite->getQueryBus()->sendWithRouting('order.getOrders'));

    $ecotoneLite->run('orders', ExecutionPollingMetadata::createWithDefaults()->withTestingSetup());
    /** Nothing should change, as we have not sent any new command message */
    $this->assertEquals(['milk'], $ecotoneLite->getQueryBus()->sendWithRouting('order.getOrders'));
}
```

1. `Class Resolver` - First argument will tell Ecotone, which classes should be resolved for this test case
2. `Dependency Container` - We define exact class instances to be used by Ecotone
3. `Clear Queue` - As we are using static name for the queue, we need to clear it to be sure nothing left from other tests
4. `Asynchronous Package` - By default asynchronous package is off for tests, this means that `Command Handler` would be called synchronously. As we want to run it asynchronously we need to turn it on
5. `Send Command` - We are using Command Bus to send our command to the Handler
6. `Run Consumer` - Ecotone for Message Channels by default registers [Polling Consumer](../../messaging-concepts/consumer.md#polling-consumer). So we can run it right away
