# Converting Results

## Converting Results

In rich business domains we will want to work with higher level objects than associate arrays. \
Suppose we have PersonNameDTO and defined [Ecotone's Converter](../../../../messaging/conversion/conversion.md) for it.

```php
class PersonNameDTOConverter
{
    #[Converter]
    public function from(PersonNameDTO $personNameDTO): array
    {
        return [
            "person_id" => $personNameDTO->getPersonId(),
            "name" => $personNameDTO->getName()
        ];
    }

    #[Converter]
    public function to(array $personNameDTO): PersonNameDTO
    {
        return new PersonNameDTO($personNameDTO['person_id'], $personNameDTO['name']);
    }
}
```

### Converting to Collection of Objects

```php
/**
* @return PersonNameDTO[]
*/
#[DbalQuery('SELECT person_id, name FROM persons LIMIT :limit OFFSET :offset')]
public function getNameListDTO(int $limit, int $offset): array;
```

Ecotone will read the Docblock and based on that will deserialize _Result Set from database_ to list of _PersonNameDTO_.

### Converting to single Object

```php
#[DbalQuery(
    'SELECT person_id, name FROM persons WHERE person_id = :personId',
    fetchMode: FetchMode::FIRST_ROW
)]
public function getNameDTO(int $personId): PersonNameDTO;
```

Using combination of First Row Fetch Mode, we can get first row and then use it for conversion to PersonNameDTO.

## Converting Iterator

For big result set we may want to avoid fetching everything at once, as it may consume a lot of memory. In those situations we may use _Iterator Fetch Mode_, to fetch one by one.\
If we want to convert each result to given Class, we may define docblock describing the result:

<pre class="language-php"><code class="lang-php">/**
 * @return iterable&#x3C;PersonNameDTO>
 */
<strong>#[DbalQuery(
</strong>    'SELECT person_id, name FROM persons ORDER BY person_id ASC',
    fetchMode: FetchMode::ITERATE
)]
public function getPersonIdsIterator(): iterable;
</code></pre>

Each returned row will be automatically convertered to _PersonNameDTO_.

## Converting to specific Media Type Format

We may return the result in specific format directly. This is useful when Business Method is used on the edges of our application and we want to return the result directly.

```php
#[DbalQuery(
    'SELECT person_id, name FROM persons WHERE person_id = :personId',
    fetchMode: FetchMode::FIRST_ROW,
    replyContentType: 'application/json'
)]
public function getNameDTOInJson(int $personId): string;
```

In this example, result will be returned in _application/json_.\
