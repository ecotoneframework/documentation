---
description: Message Driven System with Domain Driven Design principles in PHP
---

# Introduction

The foundation idea

The roots of Object Oriented Programming were mainly about communication using Messages and logic encapsulation. The aim was to focus on the flows and communication, not on the objects itself. Objects were meant to be encapsulating logic, and expose clear interfaces of what they do, and what have they done.

If you know things like Events, Commands and Aggregates, then what was written above should feel familiar to you. This is because those concepts are build around same principles of old OOP where communication is done through Messages and Objects are meant to encapsulate logic. \
And Ecotone is about returning to those roots of Object Oriented Programming. It's about explicit System design where communication happen through Messages, in a way that is clear to follow and understand.

## Message based communication

There is no possibility to immerse fully into Message based communication, as long as the foundation is not fully Message Driven. This means that each communication within the Application (not only between Applications) has to happen through Messages. This way it can become natural practice of how the system is being designed.

Ecotone follows on this making the Messaging the core of the Framework. It introduce Message based communication build around [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) as the underlying foundation. This way even communication between PHP Objects can be done through Messages in seamless way.&#x20;

<figure><img src="../.gitbook/assets/communication.png" alt=""><figcaption><p>Communication between Objects using Messages</p></figcaption></figure>

As Ecotone follows Enterprise Integration Patterns, it makes the **communication between Objects happening through Message Channels.** We can think of Message Channel as pipe, where one side send Messages into, and the other consumes from it. As communication goes through Message Channels, it becomes really easy to switch the pipes. This basically means we can easily switch Message Channel implementations to use synchronous or asynchronous Channel, different Message Brokers, and yet our Objects will not be affected by that anyhow.

## Application level code

Ecotone provides different levels of abstractions, which we can choose to use from. Each abstraction is described in more details in related sections. In this Introduction section we will  go over high level on how things can be used, to show what is Message based communication about.

### Command Handlers

Let's discuss our example from the above screenshot, where we want to register User and trigger Notification Sender. In Ecotone flows, we would introduce **Command Handler** being responsible for user registration:

```php
class UserService
{
    #[CommandHandler]
    public function register(RegisterUser $command, EventBus $eventBus): void
    {
        // store user
        
        $eventBus->publish(new UserWasRegistered($userId));
    |
}
```

As you can see, we also inject **Event Bus** which will publish our Event Message of **User Was Registered**.

{% hint style="success" %}
Command and Events are the sole of higher level Messaging. On the low level everything is Message, yet each Message can either be understood as Command (intention to do), or Event (fact that happened). This make the clear distinction between - what we want to happen vs what actually had happened.
{% endhint %}

In our Controller we would inject **Command Bus** to send the Command Message:

```php
public function registerAction(Request $request, CommandBus $commandBus): Response
{
    $command = // construct command
    $commandBus->send($command);
}
```

After sending Command Message, our Command Handler will be executed. **Command and Event Bus are available in our Dependency Container** after Ecotone installation out of the box.

{% hint style="success" %}
What is important here is that, Ecotone never forces us to implement or extend Framework specific classes. This means that our Command or Event Messages are POPO (clean PHP objects). In most of the scenarios we will simply mark given method with Attribute and Ecotone will glue the things for us.&#x20;
{% endhint %}

[Click, to find out more...](command-handling/external-command-handlers/)

### Event Handlers

We mentioned Notification Sender to be executed when User Was Registered Event happens.\
For this we follow same convention of using Attributes:

```php
class NotificationSender
{
    #[EventHandler]
    public function when(UserWasRegistered $event): void
    {
        // send notification
    }
}
```

This Event Handler will be automatically triggered when related Event will be published. \
This way we can easily build decoupled flows, which hook in into existing business events.&#x20;

{% hint style="success" %}
Even so Commands and Events are Messages at the fundamental level, Ecotone distinguish them because they carry different semantics.  By design Commands can only have single related Command Handler, yet Events can have multiple subscribing Event Handlers. \
\
This makes it easy for Developers to reason about the system and making it much easier to follow, as the difference between Messages is built in into the architecture itself.
{% endhint %}

[Click, to find out more...](command-handling/external-command-handlers/event-handling.md)

### Command Handlers with Routing

From here we could decide to make use Message routing functionality to decouple Controllers from constructing Command Messages.

```php
#[CommandHandler(routingKey: "user.register")]
public function register(RegisterUser $command, EventBus $eventBus): void
```

with this in mind, we can now user **CommandBus with routing** and even let Ecotone deserialize the Command, so our Controller does not even need to be aware of transformations:

```php
public function registerAction(Request $request, CommandBus $commandBus): Response
{
    $commandBus->sendWithRouting(
        routingKey: "user.register", 
        command: $request->getContent(), 
        commandMediaType: "application/json"
    );
}
```

{% hint style="success" %}
When controllers simply pass through incoming data to Command Bus via routing, there is not much logic left in controllers. We could even have single controller, if we would be able to get routing key.  It's really up to us, what work best in context of our system.
{% endhint %}

[Click, to find out more...](command-handling/external-command-handlers/#sending-commands-via-routing)

### Interceptors

What we could decide to do is to add so called Interceptors (middlewares) to our Command Bus to add additional data or perform validation or access checks.&#x20;

```php
#[Before(pointcut: CommandBus::class)]
public function validateAccess(
    RegisterUser $command,
    #[Reference] AuthorizationService $authorizationService
): void
{
    if (!$authorizationService->isAdmin) {
        throw new AccessDenied();
    }
}
```

Pointcut provides the point which this interceptor should trigger on. In above scenario it will trigger when Command Bus is triggered before Message is send to given Command Handler.\
The reference attribute stays that given parameter is Service from Dependency Container and Ecotone should inject it.&#x20;

{% hint style="success" %}
There are multiple different interceptors that can hook at different moments of the flow. \
We could hook before Message is sent to Asynchronous Channel, or before executing Message Handler. \
We could also state that we want to hook for all Command Handlers or Event Handlers.\
And in each step we can decide what we want to do, like modify the messages, stop the flow, enforce security checks.
{% endhint %}

[Click, to find out more...](extending-messaging-middlewares/interceptors/)

### Message Metadata

When we send Command using Command Bus, Ecotone under the hood construct a Message. \
**Message contains of two things - payload and metadata.**\
Payload is our Command and metadata is any additional information we would like to carry.

```php
public function registerAction(Request $request, CommandBus $commandBus): Response
{
    $commandBus->sendWithRouting(
        routingKey: "user.register", 
        command: $request->getContent(), 
        commandMediaType: "application/json",
        metadata: [
            "executorId" => $this->currentUser()->getId()
        ]
    );
}
```

Metadata can be easily then accessed from our Command Handler or Interceptors

```php
#[CommandHandler(routingKey: "user.register")]
public function register(
    RegisterUser $command, 
    EventBus $eventBus,
    #[Header("executorId")] string $executorId,
): void
```

Besides metadata that we do provide, Ecotone provides additional metadata that we can use whenever needed, like Message Id, Correlation Id, Timestamp etc.

{% hint style="success" %}
Ecotone take care of automatic Metadata propagation, no matter if execution synchronous or asynchronous. Therefore we can easily access any given metadata in targeted Message Handler, and also in any sub-flows like Event Handlers. This make it really easy to carry any additional information, which can not only be used in first executed Message Handler, but also in any flow triggered as a result of that.
{% endhint %}

[Click, to find out more...](extending-messaging-middlewares/message-headers.md)

### Asynchronous Processing

As we mentioned at the beginning of this introduction, communication happen through Message Channels, and thanks to that it's really easy to switch code from synchronous to asynchronous execution. For that we would simply state that given Message Handler should be executed asynchronously:

```php
#[Asynchronous("async")]
#[EventHandler]
public function when(UserWasRegistered $event): void
```

Now before this Event Handler will be executed, it **will land in Asynchronous Message Channel** named **"async"** first, and from there it will be consumed asynchronously by Message Consumer (Worker process).

{% hint style="success" %}
There maybe situations where multiple Asynchronous Event Handlers will be subscribing to same Event. \
We can easily imagine that one of them may fail and things like retries become problematic (As they may trigger successful Event Handlers for the second time).\
\
That's why Ecotone introduces [Message Handling isolation](recovering-tracing-and-monitoring/message-handling-isolation.md), which deliver a copy of the Message to each related Event Handler separately. As a result each Asynchronous Event Handler is handling it's own Message in full isolation, and in case of failure only that Handler will be retried.
{% endhint %}

[Click, to find out more...](../quick-start-php-ddd-cqrs-event-sourcing/asynchronous-php.md)

### Aggregates

If we are using Eloquent, Doctrine ORM, or Models with custom storage implementation, we could push our implementation even further and send the Command directly to our Model.&#x20;

```php
#[Aggregate]
class User
{
    use WithEvents;

    #[Identifier]
    private UserId     $userId;
    private UserStatus $status;

    #[CommandHandler(routingKey: "user.register")]
    public static function register(RegisterUser $command): self
    {
        $user = //create user
        $user->recordThat(new UserWasRegistered($userId));
        
        return $user;
    }
    
    #[CommandHandler(routingKey: "user.block")]
    public function block(): void
    {
        $this->status = UserStatus::blocked;
    }
}
```

We are marking our model as **Aggregate**, this is concept from Domain Driven Design, which describe a **model that encapsulates the business logic**.&#x20;

{% hint style="success" %}
Ecotone will take care of loading the Aggregate and storing them after the method is called. \
Therefore all we need to do it to send an Command.
{% endhint %}

Like you can see, we also added "block()" method, which will block given user. Yet it does not hold any Command as parameter. In this scenario we don't even need Command Message, because the logic is encapsulated inside nicely, and passing a status from outside could actually allow for bugs (e.g. passing UserStatus::active). Therefore all we want to know is that there is intention to block the user, the rest happens within the method.&#x20;

To execute our block method we would call Command Bus this way:

```php
public function blockAction(Request $request, CommandBus $commandBus): Response
{
    $commandBus->sendWithRouting(
        routingKey: "user.block", 
        metadata: [
            "aggregate.id" => $request->get('userId'),
        ]
    );
}
```

There is one special metadata here, which is "**aggregate.id**", this tell Ecotone the instance of User which it should fetch from storage and execute this method on. There is no need to create Command Class at all, because there is no data we need to pass there. \
This way we can build features with ease and protect internal state of our Models, so they are not modified in incorrect way.&#x20;

[Click, to find out more...](command-handling/state-stored-aggregate/)

### Workflows

One of the powers that Message Driven Architecture brings is ability to build most sophisticated workflows with ease. This is possible thanks, because each Message Handler is considered as Endpoint being able to connect to input and output Channels. **This is often referenced as pipe and filters architecture**, but in general this is characteristic of true message-driven systems.

Let's suppose that our registered user can apply for credit card, and for this we need to pass his application through series of steps to verify if it's safe to issue credit card for him:

```php
class CreditCardApplicationProcess
{
    #[CommandHandler(
        routingKey: "apply_for_card",
        outputChannelName: "application.verify_identity"
    )]
    public function apply(CardApplication $application): CardApplication
    {
        // store card application
        
        return $application;
    }
}
```

We are using **outputChannelName** here to indicate where to pass Message after it's handled by our Command Handler. In here we could enrich our CardApplication with some additional data, or create new object. However it's fully fine to pass same object to next step, if there was no need to modify it.

{% hint style="success" %}
Ecotone provides ability to pass same object between workflow steps. This simplify the flow a lot, as we are not in need to create custom objects just for the framework needs, therefore we stick what is actually needed from business perspective.
{% endhint %}

Let's define now location where our Message will land after:

```php
#[Asynchronous("async")]
#[InternalHandler(
    inputChannelName: "application.verify_identity",
    outputChannelName: "application.send_result"
)]
public function verifyIdentity(CardApplication $application): ApplicationResult
{
    // do the verification
    
    return new ApplicationResult($result);
}
```

We are using here InternalHandler, **internal handlers are not connected to any Command or Event Buses**, therefore we can use them as part of the workflow steps, which we don't want to expose outside.&#x20;

{% hint style="success" %}
It's really up to us whatever we want to define Message Handlers in separate classes or not. In general due to declarative configuration in form of Attributes, we could define the whole flow within single class, e.g. "CardApplicationProcess".\
\
Workflows can also be started from Command or Event Handlers, and also directly through Business Interfaces. This makes it easy to build and connect different flows, and even reuse steps when needed.&#x20;
{% endhint %}

Our Internal Handler contains of **inputChannelName** which points to the same channel as our Command Handlers **outputChannelName**. This way we bind Message Handlers together to create workflows. As you can see we also added Asynchronous attribute, as process of identity verification can take a bit of time, we would like it to happen in background.&#x20;

Let's define our last step in Workflow:

```php
#[InternalHandler(
    inputChannelName: "application.send_result"
)]
public function sendResult(ApplicationResult $application): void
{
    // send result
}
```

This we've made synchronous which is the default if no Asynchronous attribute is defined. Therefore it will be called directly after Identity verification.&#x20;

{% hint style="success" %}
Workflows in Ecotone are fully under our control defined in PHP. There is no need to use 3rd party, or to define the flows within XMLs or YAMLs. This makes it really maintainable solution, which we can change, modify and test them easily, as we are fully on the ownership of the process from within the code.\
\
It's worth to mention that workflows are in general stateless as they pass Messages from one pipe to another. However if we would want to introduce statefull Workflow we could do that using Ecotone's Sagas.&#x20;
{% endhint %}

[Click, to find out more...](business-workflows/)

### Inbuilt Resiliency

Ecotone handles failures at the architecture level to make Application clear of those concerns.  \
As Messages are the main component of communication between Applications, Modules or even Classes in Ecotone, it creates space for recoverability in all parts of the Application. As Messages can be retried instantly or with delay without blocking other processes from continuing their work.&#x20;

<figure><img src="../.gitbook/assets/event-handler.png" alt=""><figcaption><p>Message failed and will be retried with delay</p></figcaption></figure>

As Message are basically data records which carry the intention, it opens possibility to store that "intention", in case unrecoverable failure happen. This means that when there is no point in delayed retries, because we encountered unrecoverable error, then we can move that Message into persistent store. This way we don't lose the information, and when the bug is fixed, we can simply retry that Message to resume the flow from the place it failed.

<figure><img src="../.gitbook/assets/retry.png" alt=""><figcaption><p>Storing Message for later review, and replaying when bug is fixed</p></figcaption></figure>

There are of course more resiliency patterns, that are part of Ecotone, like:&#x20;

* Automatic retries to send Messages to Asynchronous Message Channels
* Reconnection of Message Consumers (Workers) if they lose the connection to the Broker&#x20;
* Inbuilt functionalities like Message Outbox, Error Channels with Dead Letter, Deduplication of Messages to avoid double processing,&#x20;
* and many many more.&#x20;

{% hint style="success" %}
The flow that Ecotone based on the Messages makes the Application possibile to handle failures at the architecture level. By communicating via Messages we are opening for the way, which allows us to self-heal our application without the need for us intervene, and in case of unrecoverable failures to make system robust enough to not lose any information and quickly recover from the point o failure when the bug is fixed.
{% endhint %}

[Click, to find out more...](recovering-tracing-and-monitoring/resiliency/)

## Business Oriented Architecture

Ecotone shifts the focus from technical details to the actual business processes, using **Resilient Messaging** as the foundation on which everything else is built. It provides seamless communication using Messages between Applications, Modules or even different Classes.

Together with that we will be using **Declarative Configuration** with attributes to avoid writing and maintaining configuration files. We will be stating intention of what we want to achieve instead wiring things ourselves, as a result we will regain huge amount of time, which can be invested in more important part of the System.\
\
And together with that, we will be able to use higher level **Build Blocks** like Command, Event Handlers, Aggregates, Sagas which connects to the messaging seamlessly, and helps encapsulate our business logic.\
\
So all the above serves as pillars for creating so called **Business Oriented Architecture:**

<figure><img src="../.gitbook/assets/image (4).png" alt="" width="563"><figcaption><p>When all thee pillars are solved by Ecotone, what is left to write is Business Oriented Code</p></figcaption></figure>

1. **Resilient Messaging** **-** At the heart of Ecotone lies a resilient messaging system that enables loose coupling, fault tolerance, and self-healing capabilities.
2. **Declarative Configuration -** Introduces declarative programming with Attributes. It simplifies development, reduces boilerplate code, and promotes code readability. It empowers developers to express their intent clearly, resulting in more maintainable and expressive codebases.
3. **Building Blocks -** Building blocks like Message Handlers, Aggregates, Sagas, facilitate the implementation of the business logic. By making it possible to bind Building Blocks with Resilient Messaging, Ecotone makes it easy to build and connect even the most complex business workflows.

Having this foundation knowledge and understanding how Ecotone works on the high level, it's good moment to dive into [Tutorial section](../tutorial-php-ddd-cqrs-event-sourcing/), which will provide hands on experience to deeper understanding.

## Materials

Ecotone blog provides articles which describes Ecotone's architecture and related features in more details. Therefore if you want to find out more, follow bellow links:&#x20;

### Links

* [Robust and Developer Friendly Architecture in PHP](https://blog.ecotone.tech/building-resilient-and-scalable-systems-by-default/)
* [Practical Domain Driven Design](https://blog.ecotone.tech/practial-domain-driven-design/)
* [Reactive and Message Driven Systems in PHP](https://blog.ecotone.tech/building-reactive-message-driven-systems-in-php/)

