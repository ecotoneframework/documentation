# Document Store

Document Store provides set of functionalities to store and retrieve JSON, Arrays, Objects, and Collection of Objects in a way that you are free from knowledge about storage and serialization and deserialization.&#x20;

* [DBAL Support](../modules/dbal-support.md#document-store)

## How to use

If you have installed supporting module like DBAL, then it's automatically registered in your `Depedency Container`.

```php
final class UserStore
{
    public function __construct(private DocumentStore $documentStore) {}

    public function store(User $user): void
    {
        $this->documentStore->addDocument("users", $user->getId(), $user);
    }
    
    public function getUser(int $userId): User
    {
        return $this->documentStore->getDocument("users", $userId);
    }
}
```

The first parameter, which in above example is `"users"` is collection under which users will be stored. You may use different collections for different objects to separate them. \
\
In `store method` we are passing `User` object to `addDocument` Ecotone will convert this object to `JSON` and store it chosen provider's storage.\
\
When we fetch the user using `getDocument` we are passing the saved collection name and user id, the conversion back to the user will be done automatically.&#x20;

### Storing Aggregates in your Document Store

With Dbal Document Store you can [enable State-Stored Aggregate Repository backed by Document Store](../modules/dbal-support.md#standard-aggregate-repository).

## Conversions

You may store JSON or simple arrays, objects and collection of objects (User\[]).&#x20;

{% hint style="info" %}
You will need to have Converter for JSON registered. \
If you are using `ecotone/jms-converter`, then it will be done for you.
{% endhint %}

## Other Possibles Methods

Document Store provides set of methods, like `drop whole collection`, `updating / upserting document`, `deleting` and `counting documents.`
