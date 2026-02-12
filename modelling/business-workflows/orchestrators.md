---
description: Learn how to build predefined and dynamic workflows using Orchestrator
---

# Orchestrators: Declarative Workflow Automation

While [connecting handlers with channels](connecting-handlers-with-channels.md) works great for linear workflows, and [Sagas](sagas.md) excel at stateful processes, **Orchestrator** is perfect when you need **predefined workflows** where the workflow definition is separate from the individual steps.

**You'll know you need this when:**

* You have multi-step business processes (order fulfillment, payment processing, onboarding flows) and the workflow logic is scattered across event handlers
* Business stakeholders ask "what are the steps in this process?" and the answer requires reading multiple files
* You need to add, remove, or reorder steps in a process and it touches code in many places
* Different inputs should trigger different step sequences (e.g., digital vs. physical product fulfillment)
* You want each step independently testable and reusable across different workflows

**Think of Orchestrator as**: A conductor that knows the entire symphony (workflow) and tells each musician (step) when to play, while the musicians focus only on their part.

{% hint style="info" %}
**Prerequisites**: Understanding of [message handlers](../command-handling/) and [channels](connecting-handlers-with-channels.md) will help you get the most out of Orchestrator.
{% endhint %}

{% hint style="success" %}
**Enterprise Feature**: Orchestrator is part of Ecotone's Enterprise features.
{% endhint %}

## Creating Your First Orchestrator

An Orchestrator defines a workflow as a sequence of steps (channel names) and implements those steps as internal handlers.

### Step 1: Define the Workflow

```php
class ImageProcessingOrchestrator
{
    #[Orchestrator(inputChannelName: "process.image")]
    public function processImage(): array
    {
        return [
            "resize.image",
            "add.watermark", 
            "optimize.image",
            "upload.image"
        ];
    }
}
```

**Key parts:**

* `#[Orchestrator]` - Tells Ecotone this method defines a workflow
* `inputChannelName` - Channel that triggers this workflow
* Return array - List of steps (channel names) to execute in order

### Step 2: Implement the Steps

```php
class ImageProcessingOrchestrator
{
    // ... workflow definition ...

    #[InternalHandler(inputChannelName: "resize.image")]
    public function resizeImage(ImageData $image): ImageData
    {
        // Resize logic here
        return $image->resize(800, 600);
    }

    #[InternalHandler(inputChannelName: "add.watermark")]
    public function addWatermark(ImageData $image): ImageData
    {
        // Watermark logic here
        return $image->addWatermark('© Company');
    }

    #[InternalHandler(inputChannelName: "optimize.image")]
    public function optimizeImage(ImageData $image): ImageData
    {
        // Optimization logic here
        return $image->optimize();
    }

    #[InternalHandler(inputChannelName: "upload.image")]
    public function uploadImage(ImageData $image): string
    {
        // Upload logic here
        $url = $this->storageService->upload($image);
        return $url;
    }
}
```

**What happens when you trigger the workflow:**

1. Message sent to `process.image` channel
2. Orchestrator returns `["resize.image", "add.watermark", "optimize.image", "upload.image"]`
3. Each step executes in sequence, passing data to the next step
4. Final result is returned

## Data Enrichment with Headers

Sometimes you need to add metadata or context without changing the main payload. Use `changingHeaders: true` for this:

### Enriching with Additional Data

```php
class OrderProcessingOrchestrator
{
    #[Orchestrator(inputChannelName: "process.order")]
    public function processOrder(): array
    {
        return [
            "validate.order",
            "enrich.customer.data",
            "calculate.pricing",
            "finalize.order"
        ];
    }

    #[InternalHandler(inputChannelName: "validate.order")]
    public function validateOrder(Order $order): Order
    {
        // Validation logic
        if (!$order->isValid()) {
            throw new InvalidOrderException();
        }
        return $order;
    }

    #[InternalHandler(
        inputChannelName: "enrich.customer.data",
        changingHeaders: true
    )]
    public function enrichCustomerData(Order $order): array
    {
        // Fetch additional customer data
        $customer = $this->customerService->getCustomer($order->customerId);
        $loyaltyLevel = $this->loyaltyService->getLevel($order->customerId);
        
        // Return data that becomes message headers
        return [
            'customerType' => $customer->type,
            'loyaltyLevel' => $loyaltyLevel,
            'creditScore' => $customer->creditScore
        ];
    }

    #[InternalHandler(inputChannelName: "calculate.pricing")]
    public function calculatePricing(
        Order $order,
        #[Header('customerType')] string $customerType,
        #[Header('loyaltyLevel')] string $loyaltyLevel
    ): Order {
        // Use enriched data for pricing
        $discount = $this->getDiscount($customerType, $loyaltyLevel);
        return $order->applyDiscount($discount);
    }

    #[InternalHandler(inputChannelName: "finalize.order")]
    public function finalizeOrder(Order $order): OrderConfirmation
    {
        // Final processing
        $this->orderService->finalize($order);
        
        // We don't really need to return anything, we could make the method void
        return new OrderConfirmation($order->id);
    }
}
```

**Benefits of header enrichment:**

* Keep original payload unchanged
* Add context data for downstream steps
* Maintain clean separation of concerns

## Executing Orchestrators

There are several ways to trigger orchestrator workflows:

### Method 1: Command Handler with Output Channel

```php
class ImageController
{
    #[CommandHandler('image.upload', outputChannelName: 'process.image')]
    public function uploadImage(UploadImageCommand $command): ImageData
    {
        // Handle the upload and prepare data for processing
        $imageData = $this->imageService->upload($command->file);

        // Return data that will be sent to the orchestrator
        return $imageData;
    }
}
```

**Flow:**

1. `UploadImageCommand` sent to command handler
2. Handler processes upload and returns `ImageData`
3. Result automatically sent to `process.image` channel
4. Orchestrator workflow begins

### Method 2: From Event Handlers (Business Workflows)

```php
class OrderEventHandler
{
    #[EventHandler(outputChannelName: "process.order")]
    public function whenOrderPlaced(OrderPlaced $event): Order
    {
        // Convert event to order data for processing
        return Order::fromEvent($event);
    }
}
```

**Flow:**

1. `OrderPlaced` event occurs
2. Event handler processes it and sends result to `process.order`
3. Orchestrator workflow begins automatically

### Method 3: Business Interface Triggering Business Workflow

Business Interface is simple interface where Ecotone delivers implementation. This way we can easily create and entrypoint with interface that is part of our application level code and execute the workflow:

```php
interface OrderProcessingService
{
    #[BusinessMethod(outputChannelName: 'process.order')]
    public function processOrder(Order $order): OrderConfirmation;
}
```

**Usage in your application:**

```php
class OrderController
{
    public function __construct(private OrderProcessingService $orderService) {}

    public function processOrder(Request $request): JsonResponse
    {
        $order = Order::fromRequest($request);

        // This will trigger the orchestrator workflow
        $confirmation = $this->orderService->processOrder($order);

        return new JsonResponse($confirmation);
    }
}
```

### Method 4: Custom Orchestrator Gateway

For dynamic workflows where you want to pass the steps programmatically:

```php
interface OrderProcessingGateway
{
    #[OrchestratorGateway]
    public function processWithSteps(array $steps, Order $order, array $metadata): OrderConfirmation;
}
```

```php
class OrderController
{
    public function __construct(private OrderProcessingGateway $gateway) {}

    public function processOrder(Request $request): Response
    {
        $order = Order::fromRequest($request);
        
        // Dynamically determine steps based on order type
        $steps = $this->determineStepsForOrder($order);
        
        $result = $this->gateway->processWithSteps($steps, $order, []);
        
        return new JsonResponse($result);
    }

    private function determineStepsForOrder(Order $order): array
    {
        $steps = ["validate.order"];
        
        if ($order->requiresApproval()) {
            $steps[] = "manual.approval";
        }
        
        $steps[] = "process.payment";
        
        if ($order->isInternational()) {
            $steps[] = "customs.declaration";
        }
        
        $steps[] = "ship.order";
        
        return $steps;
    }
}
```

**Gateway benefits:**

* Dynamic workflow construction
* Runtime step determination
* Easy integration with web controllers
* Flexible business rule application

## Asynchronous Orchestration

Make your workflows asynchronous for better performance and scalability:

### Asynchronous Orchestrator

```php
class AsyncImageProcessingOrchestrator
{
    #[Asynchronous('async')]
    #[Orchestrator(inputChannelName: "async.process.image")]
    public function processImageAsync(): array
    {
        return [
            "resize.image",
            "add.watermark",
            "optimize.image", 
            "upload.image"
        ];
    }

    // Steps can also be asynchronous individually
    #[Asynchronous('async')]
    #[InternalHandler(inputChannelName: "resize.image")]
    public function resizeImage(ImageData $image): ImageData
    {
        // Heavy processing that benefits from async execution
        return $this->heavyResizeOperation($image);
    }

    #[InternalHandler(inputChannelName: "add.watermark")]
    public function addWatermark(ImageData $image): ImageData
    {
        // This step runs synchronously
        return $image->addWatermark('© Company');
    }
}
```

### Mixed Synchronous/Asynchronous Steps

```php
class MixedProcessingOrchestrator
{
    #[Orchestrator(inputChannelName: "mixed.workflow")]
    public function mixedWorkflow(): array
    {
        return [
            "quick.validation",    // Sync - needs immediate feedback
            "heavy.processing",    // Async - can take time
            "send.notification"    // Async - fire and forget
        ];
    }

    #[InternalHandler(inputChannelName: "quick.validation")]
    public function quickValidation(Data $data): Data
    {
        // Fast validation that should block if it fails
        if (!$data->isValid()) {
            throw new ValidationException();
        }
        return $data;
    }

    #[Asynchronous('async')]
    #[InternalHandler(inputChannelName: "heavy.processing")]
    public function heavyProcessing(Data $data): Data
    {
        // CPU-intensive work that can be done in background
        return $this->performComplexCalculations($data);
    }

    #[Asynchronous('notifications')]
    #[InternalHandler(inputChannelName: "send.notification")]
    public function sendNotification(Data $data): void
    {
        // Fire-and-forget notification
        $this->notificationService->send($data);
    }
}
```

## Advanced Features

### Dynamic Workflow Building

The power of Orchestrator shines when you build workflows dynamically based on business rules:

```php
class DynamicCustomerOnboardingOrchestrator
{
    public function __construct(
        private CustomerService $customerService,
        private ComplianceService $complianceService
    ) {}

    #[Orchestrator(inputChannelName: "onboard.customer")]
    public function onboardCustomer(Customer $customer): array
    {
        $steps = ["validate.customer"];

        // Add steps based on customer type
        if ($customer->isEnterprise()) {
            $steps[] = "enterprise.verification";
            $steps[] = "compliance.check";
        }

        // Add steps based on location
        if ($customer->isInternational()) {
            $steps[] = "international.verification";
        }

        // Add steps based on business rules
        if ($this->complianceService->requiresKYC($customer)) {
            $steps[] = "kyc.verification";
        }

        // Common final steps
        $steps[] = "create.account";
        $steps[] = "send.welcome.email";

        return $steps;
    }
}
```

### Conditional Step Execution

Steps can return `null` to end the workflow early:

```php
class ConditionalProcessingOrchestrator
{
    #[Orchestrator(inputChannelName: "conditional.process")]
    public function conditionalProcess(): array
    {
        return [
            "check.eligibility",
            "process.if.eligible",
            "finalize.process"
        ];
    }
therefore void return type and workflow is finalized here.
    #[InternalHandler(inputChannelName: "check.eligibility")]
    public function checkEligibility(Application $application): ?Application
    {
        if (!$application->isEligible()) {
            // Returning null stops the workflow
            return null;
        }
        return $application;
    }

    #[InternalHandler(inputChannelName: "process.if.eligible")]
    public function processIfEligible(Application $application): Application
    {
        // This only runs if previous step didn't return null
        return $this->processApplication($application);
    }

    #[InternalHandler(inputChannelName: "finalize.process")]
    public function finalizeProcess(Application $application): ApplicationResult
    {
        return new ApplicationResult($application);
    }
}
```

### Nested Orchestrators

Orchestrators can call other orchestrators as steps:

```php
class MasterOrchestrator
{
    #[Orchestrator(inputChannelName: "master.workflow")]
    public function masterWorkflow(): array
    {
        return [
            "prepare.data",
            "sub.workflow.a",  // This calls another orchestrator
            "sub.workflow.b",  // This calls another orchestrator
            "combine.results"
        ];
    }

    #[InternalHandler(inputChannelName: "prepare.data")]
    public function prepareData(RawData $data): ProcessedData
    {
        return new ProcessedData($data);
    }

    #[Orchestrator(inputChannelName: "sub.workflow.a")]
    public function subWorkflowA(): array
    {
        return ["step.a1", "step.a2"];
    }

    #[Orchestrator(inputChannelName: "sub.workflow.b")]
    public function subWorkflowB(): array
    {
        return ["step.b1", "step.b2"];
    }

    #[InternalHandler(inputChannelName: "combine.results")]
    public function combineResults(ProcessedData $data): FinalResult
    {
        return new FinalResult($data);
    }
}
```

## Testing Orchestrators

Testing orchestrators is straightforward with Ecotone Lite. You can test the entire workflow, individual steps, or specific scenarios.

```php
use Ecotone\Lite\EcotoneLite;
use PHPUnit\Framework\TestCase;

class ImageProcessingOrchestratorTest extends TestCase
{
    private EcotoneLite $ecotoneLite;

    protected function setUp(): void
    {
        $this->ecotoneLite = EcotoneLite::bootstrapFlowTesting(
            [ImageProcessingOrchestrator::class],
            [
                'storageService' => new InMemoryStorageService(),
                'imageProcessor' => new TestImageProcessor()
            ],
            ServiceConfiguration::createWithDefaults()
                ->withLicenceKey(VALID_LICENCE)
        );
    }

    public function test_complete_image_processing_workflow(): void
    {
        // Arrange
        $imageData = new ImageData('test-image.jpg', 1920, 1080);

        // Act
        $result = $this->ecotoneLite->sendDirectToChannel('process.image', $imageData);

        // Assert
        $this->assertInstanceOf(ImageData::class, $result);
        $this->assertEquals(800, $result->width);
        $this->assertEquals(600, $result->height);
        $this->assertTrue($result->hasWatermark());
        $this->assertTrue($result->isOptimized());
    }
}
```

### Testing Individual Steps

```php
public function test_individual_steps(): void
{
    $imageData = new ImageData('test.jpg', 1920, 1080);

    // Test resize step
    $resized = $this->ecotoneLite->sendDirectToChannel('resize.image', $imageData);
    $this->assertEquals(800, $resized->width);
    $this->assertEquals(600, $resized->height);

    // Test watermark step
    $watermarked = $this->ecotoneLite->sendDirectToChannel('add.watermark', $resized);
    $this->assertTrue($watermarked->hasWatermark());

    // Test optimization step
    $optimized = $this->ecotoneLite->sendDirectToChannel('optimize.image', $watermarked);
    $this->assertTrue($optimized->isOptimized());
}
```

### Testing Data Enrichment

```php
public function test_order_processing_with_header_enrichment(): void
{
    $order = new Order('123', 'customer-456', [new Item('product', 100)]);

    $result = $this->ecotoneLite->sendDirectToChannel('process.order', $order);

    $this->assertInstanceOf(OrderConfirmation::class, $result);
    $this->assertTrue($result->hasDiscount()); // Discount applied based on enriched data
}

public function test_customer_data_enrichment_step(): void
{
    $order = new Order('123', 'premium-customer', []);

    // Test the enrichment step directly
    $enrichedHeaders = $this->ecotoneLite->sendDirectToChannel('enrich.customer.data', $order);

    $this->assertEquals('premium', $enrichedHeaders['customerType']);
    $this->assertEquals('gold', $enrichedHeaders['loyaltyLevel']);
    $this->assertGreaterThan(700, $enrichedHeaders['creditScore']);
}
```

### Testing Asynchronous Orchestrators

```php
public function test_asynchronous_orchestrator(): void
{
    $ecotoneLite = EcotoneLite::bootstrapFlowTesting(
        [AsyncImageProcessingOrchestrator::class],
        ['imageProcessor' => new TestImageProcessor()],
        ServiceConfiguration::createWithDefaults()
            ->withSkippedModulePackageNames(ModulePackageList::allPackagesExcept([ModulePackageList::CORE_PACKAGE, ModulePackageList::ASYNCHRONOUS_PACKAGE]))
            ->withLicenceKey(VALID_LICENCE),
        enableAsynchronousProcessing: [
            SimpleMessageChannelBuilder::createQueueChannel('async'),
        ]
    );

    $imageData = new ImageData('test.jpg', 1920, 1080);

    // Send to async workflow
    $ecotoneLite->sendDirectToChannel('async.process.image', $imageData);

    // Run async processing
    $ecotoneLite->run('async');

    // Verify processing completed
    $this->assertTrue(true); // Add specific assertions based on your implementation
}
```

### Testing Orchestrator Gateways

```php
public function test_orchestrator_gateway_with_dynamic_steps(): void
{
    $ecotoneLite = EcotoneLite::bootstrapFlowTesting(
        [OrderProcessingOrchestrator::class, OrderProcessingGateway::class],
        [new OrderProcessingOrchestrator()],
        ServiceConfiguration::createWithDefaults()
            ->withSkippedModulePackageNames(ModulePackageList::allPackagesExcept([ModulePackageList::CORE_PACKAGE]))
            ->withLicenceKey(VALID_LICENCE)
    );

    /** @var OrderProcessingGateway $gateway */
    $gateway = $ecotoneLite->getGateway(OrderProcessingGateway::class);

    $order = new Order('123', 'customer-456', []);
    $steps = ['validate.order', 'process.payment', 'ship.order'];

    $result = $gateway->processWithSteps($steps, $order, []);

    $this->assertInstanceOf(OrderConfirmation::class, $result);
}
```

## Key Benefits of Orchestrator

### 🎯 **Separation of Concerns**

* **Workflow definition** is separate from **step implementation**
* Easy to understand the entire process at a glance
* Steps can be reused across different workflows

### 🔄 **Reusability**

* Same steps can be used in multiple workflows
* Build libraries of reusable business operations
* Mix and match steps for different scenarios

### ⚡ **Dynamic Workflows**

* Build workflows programmatically based on business rules
* Adapt to different customer types, regions, or conditions
* Runtime workflow construction

### 🧪 **Testability**

* Test entire workflows end-to-end
* Test individual steps in isolation
* Easy mocking and stubbing of dependencies

### 📈 **Scalability**

* Asynchronous execution support
* Individual steps can be scaled independently
* Easy to add new steps without changing existing code

### 🔍 **Observability**

* Clear workflow execution path
* Easy to monitor and debug
* Step-by-step execution tracking

## Summary

Orchestrator is perfect for building **predefined workflows** where you want to:

* 🎯 **Separate workflow definition from step implementation**
* 🔄 **Reuse steps across different workflows**
* ⚡ **Build dynamic workflows based on business rules**
* 🧪 **Test workflows and steps independently**
* 📋 **Execute consistent, repeatable business processes**

{% hint style="success" %}
**Key insight**: Orchestrator shines when you know the types of workflows you need but want flexibility in how they're constructed and executed. It's the perfect balance between structure and flexibility.
{% endhint %}

The power of Orchestrator lies in its ability to make complex business workflows **simple to define**, **easy to test**, and **flexible to modify**. Whether you're processing orders, onboarding customers, or handling document workflows, Orchestrator provides the structure and flexibility you need to build robust, maintainable business processes.
