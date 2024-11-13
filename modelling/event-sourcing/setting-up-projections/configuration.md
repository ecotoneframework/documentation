# Configuration

## Setting up Projections

**Projections are about deriving current state from the stream of events**.\
Projections can be added in any moment of the application lifetime and be built up from existing history, till the current time. \
This is powerful concept as it allow for building views quickly and discarding them without pain, when they are not longer needed.&#x20;

```php
#[Projection("inProgressTicketList", Ticket::class] // 1
class InProgressTicketList
{
    public function __construct(private Connection $connection) {}

    #[EventHandler] // 2
    public function addTicket(TicketWasRegistered $event, array $metadata) : void
    {
        $result = $this->connection->executeStatement(<<<SQL
    INSERT INTO in_progress_tickets VALUES (?,?)
SQL, [$event->getTicketId(), $event->getTicketType()]);
    }

    #[QueryHandler("getInProgressTickets")] // 3
    public function getTickets() : array
    {
        return $this->connection->executeQuery(<<<SQL
    SELECT * FROM in_progress_tickets
SQL)->fetchAllAssociative();
    }    
}
```

1. This tells `Ecotone` that specific class is Projection. The first parameter is the `name of the projection` and the second is name of the stream (the default is the `name of the Aggregate`) that this projection subscribes to.&#x20;
2. Events that this projection subscribes to
3. Optional Query Handlers for this projection. They can be placed in different classes depending on preference.&#x20;

## Document Store

[Document Store](configuration.md#document-store) is a great way to set up your projections. \
You can freely create DTO objects, or play with simple arrays and Ecotone will serialize/deserialize and store them for you.&#x20;

```php
#[Projection(self::NAME, Account::class)]
final class AvailableBalanceProjection
{
    const NAME = "available_balance";
    const QUERY = "getCurrentBalance";

    public function __construct(private DocumentStore $documentStore) {}

    #[EventHandler]
    public function whenAccountSetup(AccountSetup $event): void
    {
        $this->documentStore->addDocument(
            self::NAME,
            $event->accountId,
            ["balance" => 0]
        );
    }

    #[EventHandler]
    public function whenPaymentMade(PaymentMade $event): void
    {
        $this->documentStore->updateDocument(
            self::NAME,
            $event->accountId,
            ["balance" => $this->getCurrentBalance($event->accountId) + $event->amount]
        );
    }

    #[QueryHandler(self::QUERY)]
    public function getCurrentBalance(UuidInterface $accountId): int
    {
        return $this->documentStore->getDocument(self::NAME, $accountId)["balance"];
    }
}
```

{% hint style="info" %}
For testing purposes you may use [In Memory implementation](../../../modules/dbal-support.md#in-memory-document-store), to speed up your tests or to simple test things out locally.&#x20;
{% endhint %}
