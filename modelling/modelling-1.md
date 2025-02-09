---
description: Message Driven System with Domain Driven Design principles in PHP
---

# Introduction

## The Main idea

The roots of Object Oriented Programming were mainly about communication using Messages and logic encapsulation. It was meant to focus on the flows and communication, not on the objects itself. Objects were meant to be encapsulating logic, and expose clear interfaces of what they do, and what have they done.

If you know things like Events or Commands and Aggregates, which are actually higher level concepts, then what was written above should feel familiar to you. This is because those concepts are build around same principles of old OOP where communicate is done through Messages. \
And this is what Ecotone is about, it's about returning to the roots of OOP, about explicit system design, where the main role are playing Messages, doing communication between Objects.&#x20;

## Message based communication

There is no possibility to immerse fully into Message based communication, as long as the foundation is not fully Message Driven. This means that each communication has to happen through Messages, this way it can become natural way of designing system and communication.&#x20;

Ecotone does that by introducing Message based communication build using [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) as the sole foundation. This way even communication between Objects within the Application can be done through Messages in seamless way. \
**Messages between Objects are send through Message Channels:**

<figure><img src="../.gitbook/assets/communication.png" alt=""><figcaption><p>Communication between Objects using Messages</p></figcaption></figure>

As communication goes through Message Channels, it become really easy to switch it to become synchronous or asynchronous, to use different Message Broker as implementation or to publish it to multiple parties. What is important is that from application level perspective, code is free of any messaging concerns, yet we are able to use all advantages it provides.

## Application level code

Ecotone provides different levels of abstractions and it's up to us, what level of abstraction we feel more comfortable to work with. Each abstraction is described in more details in related section, and we will have a chance to understand those concepts deeper in [tutorial section](../tutorial-php-ddd-cqrs-event-sourcing/). Therefore in here we will  go over high level of things on how things look like.

### Command Handlers

Suppose our example from the above, where we want to register User and trigger Notification Sender.\
In Ecotone flows, we would introduce **Command Handler** being responsible for user registration:

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
Command and Events are the sole of higher level Messaging. On the low level everything is Message, yet each Message can either be understood as Command (intention to do), or Event (fact that happened). This make the clear distinction from what we want to happen, from what already has happened.
{% endhint %}

In our Controller we would inject **Command Bus** to send the Command Message:

```php
public function registerAction(Request $request, CommandBus $commandBus): Response
{
    $command = // construct command
    $commandBus->send($command);
}
```

After sending Command Message, our Command Handler will be executed. **Command and Event Bus is available in our Dependency Container** after Ecotone installation out of the box.

{% hint style="success" %}
What is important here is that, Ecotone never forces to implement or extend Framework specific classes. This means that our Command or Event Messages are POPO (clean PHP object). In most of the scenarios we will simply mark given method with Attribute and Ecotone will glue the things for us.&#x20;
{% endhint %}

### Command Handlers with Routing

From here we could decide to make use of higher level abstraction like Message routing.

```php
#[CommandHandler("user.register")]
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

This Event Handle will be automatically triggered when related Event will be published. \
As we mentioned at the beginning of this introduction, communication happen through Message Channels, and it's really easy to switch code from synchronous to asynchronous. For that we would simply state that this Handler should be asynchronous:

```php
#[Asynchronous("async")]
#[EventHandler]
public function when(UserWasRegistered $event): void
```

Now before this Event Handler will be executed, it **will land in Asynchronous Message Channel** named "async" first.&#x20;

{% hint style="success" %}
There maybe multiple Event Handlers subscribing to same Event Message. \
We can easily imagine that some of them may fail and things like retries become problematic (As they may trigger successful Event Handlers for the second time).\
\
That's why Ecotone deliver a copy of the Message to each related Event Handler. \
As a result each Asynchronous Event Handler is handling it's own Message in full isolation, and in case of failure only that Handler will be retried.
{% endhint %}

### Aggregates

If we are using Eloquent, Doctrine ORM, or Models with our storage implementation, we could push this even further and send the Command directly to our Model.&#x20;

```php
#[Aggregate]
class User
{
    use WithEvents;

    #[Identifier]
    private UserId     $userId;
    private UserStatus $status;

    #[CommandHandler("user.register")]
    public static function register(RegisterUser $command): self
    {
        $user = //create user
        $user->recordThat(new UserWasRegistered($userId));
        
        return $user;
    }
    
    #[CommandHandler("user.block")]
    public function block(): void
    {
        $this->status = UserStatus::blocked;
    }
}
```

We are marking our model as **Aggregate**, this is concept from Domain Driven Design, which describe a **model that encapsulates the business logic**. Like you can, we also added block method, we can know send an Command which will block the user. This way we can simply build explicit models that protect internal state from being modified in incorrect way.&#x20;

{% hint style="success" %}
Ecotone will take care of loading the Aggregate and storing them after the method is called. \
Therefore all we need to do it to send an Command.
{% endhint %}

Like you can see this all works in seamless way, we are working with Message Driven Architecture, yet the code we write is free of any Message concerns. We simply use PHP classes that we've created and Ecotone glues everything around. This is possible thanks to Ecotone's architecture that is built on three main pillars.

## Business Oriented Architecture

[Ecotone](https://blog.ecotone.tech/revolutionary-boa-framework-ecotone/) embrace the concept of Business-Oriented Architecture, which follows fundamental principle of making business logic the primary citizen in our Applications. It shifts the focus from technical details to the actual business processes. \
**Business Oriented Architecture** is built around three main pillars:

<figure><img src="../.gitbook/assets/image (4).png" alt="" width="563"><figcaption><p>When all thee pillars are solved by Ecotone, what is left to write is Business Oriented Code</p></figcaption></figure>

1. **Resilient Messaging** **-** At the heart of Ecotone lies a resilient messaging system that enables loose coupling, fault tolerance, and self-healing capabilities.
2. **Declarative Configuration -** Introduces declarative programming which simplifies development, reduces boilerplate code, and promotes code readability. It empowers developers to express their intent clearly, resulting in more maintainable and expressive codebases.
3. **Building Blocks -** Building blocks like Message Handlers, Aggregates, Sagas, facilitate the implementation of the business logic. By making it possible to bind Building Blocks with Resilient Messaging, Ecotone makes it easy to build and connect even the most complex business workflows.

By providing all those three Pillars, Ecotone provides an foundation which then allows us to fully focus on the business side of the things.

## Materials

Ecotone blog provides articles which describes Ecotone's architecture and related features in more details. Therefore if you want to find out more, follow bellow links:&#x20;

### Links

* [Robust and Developer Friendly Architecture in PHP](https://blog.ecotone.tech/building-resilient-and-scalable-systems-by-default/)
* [Practical Domain Driven Design](https://blog.ecotone.tech/practial-domain-driven-design/)
* [Reactive and Message Driven Systems in PHP](https://blog.ecotone.tech/building-reactive-message-driven-systems-in-php/)

