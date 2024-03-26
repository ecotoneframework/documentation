# Working with Database

Ecotone allows to work with Database using _DbalBusinessMethod_. The goal is to create abstraction which significantly reduce the amount of boilerplate code required to implement data access layers.\
Thanks to Dbal based Business Methods we are able to avoid writing integration and transformation level code and focus on the Business part of the system.\
\
To make use of Dbal based Business Method, [install Dbal Module first](../../../../modules/dbal-support.md).

## Write Business Methods

Let's consider scenario where we want to store new record in _Persons table_. To make it happen just like with _Business Method_ we will create an Interface, yet this time we will mark it with _DbalBusinessMethod_.

```php
interface PersonApi
{
    #[DbalWrite("INSERT INTO persons VALUES (:personId, :name)")]
    public function register(int $personId, string $name): void;
}
```

The first parameter passed to _DbalBusinessMethod_ is actual SQL, where we can provide set of named parameters. Ecotone will automatically bind parameters from method declaration to SQL ones by names.&#x20;

{% hint style="success" %}
Above example will use DbalConnectionFactory::class for database Connection, which is the default for [Dbal Module](../../../../modules/dbal-support.md). If you want to run Business Method on different connection, you can do it using _connectionReferenceName_ parameter inside the Attribute.
{% endhint %}

### Custom Parameter Name

We may bind parameter name explicitly by using DbalParameter attribute.

```php
#[DbalWrite('INSERT INTO persons VALUES (:personId, :name)')]
public function register(
    #[DbalParameter(name: 'personId')] int $id,
    string $name
): void;
```

This can be used when we want to decouple interface parameter names from binded parameters or when name in database column is not explicit enough for being part of interface.

### Returning number of records changed

If we want to return amount of the records that have been changed, we can add int type hint to our Business Method:

```php
#[DbalWrite('UPDATE persons SET name = :name WHERE person_id = :personId')]
public function changeName(int $personId, string $name): int;
```

## Query Business Methods

We may want to fetch data from the database and for this we will be using _DbalQueryBusinessMethod_.

```php
interface PersonApi
{
    #[DbalQueryMethod('SELECT person_id, name FROM persons LIMIT :limit OFFSET :offset')]
    public function getNameList(int $limit, int $offset): array;
}
```

The above will return result as associative array with the columns provided in SELECT statement.

## Fetching Mode

To format result differently we may use different fetch modes. The default fetch Mode is associative array.

### First Column Fetch Mode

```php
/**
 * @return int[]
 */
#[DbalQuery(
    'SELECT person_id FROM persons ORDER BY person_id ASC LIMIT :limit OFFSET :offset',
    fetchMode: FetchMode::FIRST_COLUMN
)]
public function getPersonIds(int $limit, int $offset): array;
```

This will extract the first column from each row, which allows us to return array of person Ids directly.

### First Column of first row Mode

To get single variable out of Result Set we can use First Column of first row Mode.

```php
#[DbalQuery(
    'SELECT COUNT(*) FROM persons',
    fetchMode: FetchMode::FIRST_COLUMN_OF_FIRST_ROW
)]
public function countPersons(): int;
```

This way we can provide simple interfaces for things Aggregate SQLs, like SUM or COUNT.

### First Row Mode

To fetch first Row of given Result Set, we can use First Row Mode.

```php
#[DbalQuery(
    'SELECT person_id, name FROM persons WHERE person_id = :personId',
    fetchMode: FetchMode::FIRST_ROW
)]
public function getNameDTO(int $personId): array;
```

This will return array containing person\_id and name.

### Returning Nulls

When using First Row Mode, we may end up having no returned row at all. In this situation Dbal will return _false,_ however if Return Type will be nullable, then Ecotone will convert false to _null_.

```php
#[DbalQuery(
    'SELECT person_id, name FROM persons WHERE person_id = :personId',
    fetchMode: FetchMode::FIRST_ROW
)]
public function getNameDTOOrNull(int $personId): PersonNameDTO|null;
```

## Returning Iterator

For big result set we may want to avoid fetching everything at once, as it may consume a lot of memory. In those situations we may use _Iterator Fetch Mode_, to fetch one by one.

```php
#[DbalQuery(
    'SELECT person_id, name FROM persons ORDER BY person_id ASC',
    fetchMode: FetchMode::ITERATE
)]
public function getPersonIdsIterator(): iterable;
```

## Parameter Types

Each parameter may have different type and Ecotone will try to recognize specific type and set it up accordingly. If we want, we can take over and define the type explicitly.

```php
#[DbalQuery('SELECT * FROM persons WHERE person_id IN (:personIds)')]
public function getPersonsWith(
    #[DbalParameter(type: ArrayParameterType::INTEGER)] array $personIds
): array;
```
