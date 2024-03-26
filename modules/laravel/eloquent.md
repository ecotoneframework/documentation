# Eloquent

Ecotone comes with out of the box integration with Eloquent for [State-Stored Aggregates](../../modelling/command-handling/state-stored-aggregate/#state-stored-aggregate).\
If you've installed Laravel, then you may start using Eloquent based Aggregates.

## Your Models as Aggregates

Mark your Models with Aggregate attribute and set up Command Handlers.

```php
#[Aggregate]
final class Issue extends Model
{
    use WithEvents;

    #[CommandHandler()] // 1. issue.report factory method
    public static function reportNew(ReportIssue $command): self
    {
        $issue = new self();
        $issue->email = $command->email->address;
        $issue->open = true;
        $issue->content = $command->content;
        $issue->save(); // 3. Saving

        // 4. Event publishing
        $issue->recordThat(new IssueWasReported($issue->id));

        return $issue;
    }
    
    #[CommandHandler("issue.close")] // 2. issue.close action method
    public function close(): void
    {
       if (!$this->open) {
           return;  
       }
      
       $this->open = false;
      
       $this->recordThat(new IssueWasClosed($this->id));
    }

    #[AggregateIdentifierMethod("id")]
    public function getId(): ?int
    {
        return $this->id;
    }
}
```

1. Calling factory method **"issue.report":**

```php
$this->commandBus->send(new ReportIssue());
```

2. Calling action method **"issue.close":**

```php
$this->commandBus->sendWithRouting("issue.close", metadata: ["aggregate.id" => $id]);
```

3. Aggregates require state to be always valid. If we have auto-generated identifiers from database, then in order to be assured that returned `Issue` has id, we need to call `save`.\
   If you generate identifiers outside of the database, this step is not needed.
4. Event publishing - if we have imported trait `WithEvents`, then we can publish events from the Aggregate using _recordThat_ method.

{% hint style="success" %}
In case of Ecotone you may use routing for your Message Handlers or direct Message Classes. It's up to you to decide whatever works best in your context.
{% endhint %}

### Demo

Read more about integration in [following blog post](https://blog.ecotone.tech/build-laravel-application-using-ddd-and-cqrs/).
