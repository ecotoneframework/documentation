---
description: PHP Event Sourcing Projections with Document Store
---

# Projections with Document Store

## The Problem

You want to build a Read Model quickly but writing raw SQL for every projection — `CREATE TABLE`, `INSERT`, `UPDATE`, `SELECT` — is tedious and error-prone. You just want to store and retrieve PHP objects or arrays without managing schema yourself. How do you build projections without writing SQL?

## What is the Document Store?

Ecotone's `DocumentStore` is a key-value store that automatically serializes and deserializes PHP objects and arrays to JSON. You organize data in **collections** (like database tables) and access individual **documents** by ID.

It's available out of the box with DBAL — no extra setup needed. Think of it as a simpler alternative to writing raw SQL for your Read Models.

## Building a Projection with Document Store

Instead of injecting a `Connection` and writing SQL, inject `DocumentStore` and work with PHP objects directly:

```php
#[ProjectionV2('available_balance')]
#[FromAggregateStream(Account::class)]
class AvailableBalanceProjection
{
    public function __construct(private DocumentStore $documentStore) {}

    #[EventHandler]
    public function whenAccountSetup(AccountSetup $event): void
    {
        $this->documentStore->addDocument(
            'available_balance',
            $event->accountId,
            ['balance' => 0]
        );
    }

    #[EventHandler]
    public function whenPaymentMade(PaymentMade $event): void
    {
        $current = $this->documentStore->getDocument(
            'available_balance',
            $event->accountId
        );

        $this->documentStore->updateDocument(
            'available_balance',
            $event->accountId,
            ['balance' => $current['balance'] + $event->amount]
        );
    }

    #[QueryHandler('getCurrentBalance')]
    public function getCurrentBalance(string $accountId): int
    {
        return $this->documentStore->getDocument(
            'available_balance',
            $accountId
        )['balance'];
    }
}
```

Notice there's no `#[ProjectionInitialization]` to create tables, no `#[ProjectionDelete]` to drop them — the Document Store handles storage automatically.

## Available Operations

The `DocumentStore` interface provides these methods:

| Method | Description |
|--------|-------------|
| `addDocument($collection, $id, $document)` | Add a new document. Throws if ID already exists. |
| `updateDocument($collection, $id, $document)` | Update existing document. Throws `DocumentNotFound` if missing. |
| `upsertDocument($collection, $id, $document)` | Add or update — inserts if new, updates if exists. |
| `deleteDocument($collection, $id)` | Delete a document by ID. |
| `getDocument($collection, $id)` | Get document. Throws `DocumentNotFound` if missing. |
| `findDocument($collection, $id)` | Get document. Returns `null` if missing (no exception). |
| `getAllDocuments($collection)` | Get all documents in a collection. |
| `countDocuments($collection)` | Count documents in a collection. |
| `dropCollection($collection)` | Drop entire collection. |

## Storing PHP Objects

The Document Store can store PHP objects directly — Ecotone automatically serializes them to JSON and deserializes them back:

```php
class WalletBalance
{
    public function __construct(
        public readonly string $walletId,
        public readonly int $currentBalance,
    ) {}
    
    public function add(int $amount): self
    {
        return new self($this->walletId, $this->currentBalance + $amount);
    }
}

#[ProjectionV2('wallet_balance')]
#[FromAggregateStream(Wallet::class)]
class WalletBalanceProjection
{
    public function __construct(private DocumentStore $documentStore) {}

    #[EventHandler]
    public function whenWalletCreated(WalletWasCreated $event): void
    {
        $this->documentStore->addDocument(
            'wallet_balance',
            $event->walletId,
            new WalletBalance($event->walletId, 0)
        );
    }

    #[EventHandler]
    public function whenMoneyAdded(MoneyWasAddedToWallet $event): void
    {
        /** @var WalletBalance $wallet */
        $wallet = $this->documentStore->getDocument('wallet_balance', $event->walletId);

        $this->documentStore->updateDocument(
            'wallet_balance',
            $event->walletId,
            $wallet->add($event->amount)
        );
    }

    #[QueryHandler('getWalletBalance')]
    public function getBalance(string $walletId): WalletBalance
    {
        return $this->documentStore->getDocument('wallet_balance', $walletId);
    }
}
```

{% hint style="success" %}
When storing objects, Ecotone uses the configured serializer (e.g., JMS Converter) to convert them to JSON. The same object type is returned when reading — no manual deserialization needed.
{% endhint %}

## Using upsertDocument for Simpler Logic

When you don't want to distinguish between "first time" and "update", use `upsertDocument` to simplify your handlers:

```php
#[EventHandler]
public function whenTicketRegistered(TicketWasRegistered $event): void
{
    $this->documentStore->upsertDocument(
        'ticket_list',
        $event->ticketId,
        ['ticketId' => $event->ticketId, 'type' => $event->type, 'status' => 'open']
    );
}

#[EventHandler]
public function whenTicketClosed(TicketWasClosed $event): void
{
    $this->documentStore->upsertDocument(
        'ticket_list',
        $event->ticketId,
        ['ticketId' => $event->ticketId, 'status' => 'closed']
    );
}
```

## Lifecycle with Document Store

When using Document Store, you can simplify your lifecycle hooks by operating on collections:

```php
#[ProjectionDelete]
public function delete(): void
{
    $this->documentStore->dropCollection('wallet_balance');
}

#[ProjectionReset]
public function reset(): void
{
    $this->documentStore->dropCollection('wallet_balance');
}
```

## Testing with In-Memory Document Store

For tests, Ecotone provides an `InMemoryDocumentStore` that works identically to the DBAL version but stores everything in memory — no database needed:

```php
$ecotone = EcotoneLite::bootstrapFlowTestingWithEventStore(
    classesToResolve: [WalletBalanceProjection::class, Wallet::class],
    containerOrAvailableServices: [
        new WalletBalanceProjection(InMemoryDocumentStore::createEmpty()),
    ]
);

// Send commands, trigger projection, then query
$ecotone->sendCommand(new CreateWallet('wallet-1'));
$ecotone->sendCommand(new AddMoney('wallet-1', 100));

$balance = $ecotone->sendQueryWithRouting('getWalletBalance', 'wallet-1');
// $balance->currentBalance === 100
```

{% hint style="success" %}
`InMemoryDocumentStore` is perfect for unit and integration tests — it has the same API as the DBAL version, runs instantly, and requires no database setup.
{% endhint %}

## When to Use Document Store vs Raw SQL

| | Document Store | Raw SQL (Connection) |
|---|---|---|
| **Setup effort** | Minimal — no schema management | Requires `CREATE TABLE`, migrations |
| **Query flexibility** | Key-value only (by ID, by collection) | Full SQL (JOINs, WHERE, aggregations) |
| **Best for** | Simple Read Models, rapid prototyping | Complex queries, reporting, dashboards |
| **Lifecycle hooks** | `dropCollection()` | `CREATE TABLE` / `DROP TABLE` / `DELETE FROM` |
| **Testing** | `InMemoryDocumentStore` — no DB needed | Requires test database |

{% hint style="info" %}
You can mix both approaches in the same application — use Document Store for simple projections and raw SQL for complex ones. They are not mutually exclusive.
{% endhint %}
