# Converting Parameters

We may want to use higher level object within our Interface than simple scalar types. As those can't be understood by our Database, it means we need Conversion. Ecotone provides default conversions and possibility to customize the process.

## Default Date Time Conversion

Ecotone provides inbuilt Conversion for Date Time based objects.&#x20;

```php
#[DbalWrite('INSERT INTO activities VALUES (:personId, :time)')]
public function add(string $personId, \DateTimeImmutable $time): void;
```

By default Ecotone will convert time using `Y-m-d H:i:s.u` format. We may override this using [Custom Converters](../../../../messaging/conversion/conversion.md).

## Default Class Conversion

If your Class contains `__toString` method, it will be used for doing conversion.

```php
#[DbalWrite('INSERT INTO activities VALUES (:personId, :time)')]
public function store(PersonId $personId, \DateTimeImmutable $time): void;
```

```php
final readonly class PersonId
{
    public function __construct(private string $id) {}

    public function __toString(): string
    {
        return $this->id;
    }
}
```

We may override this using [Custom Converters](../../../../messaging/conversion/conversion.md).

## Converting Array to JSON

For example database column may be of type JSON or Binary.\
In those situation we may state what Media Type given parameter should be converted too, and Ecotone will do the conversion before it's executing SQL.

```php
 /**
  * @param string[] $roles
  */
 #[DbalWrite('UPDATE persons SET roles = :roles WHERE person_id = :personId')]
 public function changeRoles(
     int $personId,
     #[DbalParameter(convertToMediaType: MediaType::APPLICATION_JSON)] array $roles
 ): void;
```

In above example roles will be converted to JSON before SQL will be executed.

### Value Objects Conversion

If we are using higher level classes like Value Objects, we will be able to change the type to expected one. \
For example if we are using [JMS Converter Module](../../../../modules/jms-converter.md) we can register Converter for our _PersonRole_ Class and convert it to JSON or XML.

```php
final class PersonRoleConverter
{
    #[Converter]
    public function from(PersonRole $personRole): string
    {
        return $personRole->getRole();
    }
    
    #[Converter]
    public function to(string $role): PersonRole
    {
        return new PersonRole($role);
    }
}
```

{% hint style="info" %}
Read more about Ecotone's [in Converters related section.](../../../../messaging/conversion/conversion.md)
{% endhint %}

Then we will be able to use our Business Method with PersonRole, which will be converted to given Media Type before being saved:

```php
 /**
  * @param PersonRole[] $roles
  */
 #[DbalWrite('UPDATE persons SET roles = :roles WHERE person_id = :personId')]
 public function changeRolesWithValueObjects(
     int $personId,
     #[DbalParameter(convertToMediaType: MediaType::APPLICATION_JSON)] array $roles
 ): void;
```

This way we can provide higher level classes, keeping our Interface as close as it's needed to our business model.

## Using Expression Language

### Calling Method Directly on passed Object

We may use Expression Language to dynamically evaluate our parameter.

```php
#[DbalWrite('INSERT INTO persons VALUES (:personId, :name)')]
public function register(
    int $personId,
    #[DbalParameter(expression: 'payload.toLowerCase()')] PersonName $name
): void;
```

_**payload**_ is special parameter in expression, which targets value of given parameter, in this example it will be _PersonName_. \
In above example before storing **name** in database, we will call _**toLowerCase()**_ method on it.

### Using External Service for evaluation

We may also access any Service from our Dependency Container and run a method on it.

```php
#[DbalWrite('INSERT INTO persons VALUES (:personId, :name)')]
public function insertWithServiceExpression(
    int $personId,
    #[DbalParameter(expression: "reference('converter').normalize(payload)")] PersonName $name
): void;
```

_**reference**_ is special function within expression which allows us to fetch given Service from Dependency Container. In our case we've fetched Service registered under "_converter" id_ and ran _normalize_ method passing _PersonName_.

## Using Method Level Dbal Parameters

We may use Dbal Parameters on the Method Level, when parameter is not needed.

### Static Values

In case parameter is a static value.

```php
#[DbalWrite('INSERT INTO persons VALUES (:personId, :name, :roles)')]
#[DbalParameter(name: 'roles', expression: "['ROLE_ADMIN']", convertToMediaType: MediaType::APPLICATION_JSON)]
public function registerAdmin(int $personId, string $name): void;
```

### Dynamic Values

We can also use dynamically evaluated parameters and access Dependency Container to get specific Service.

```php
#[DbalWrite('INSERT INTO persons VALUES (:personId, :name, :registeredAt)')]
#[DbalParameter(name: 'registeredAt', expression: "reference('clock').now()")]
public function registerAdmin(int $personId, string $name): void;
```

### Dynamic Values using Parameters

In case of Method and Class Level Dbal Parameters we get access to passed parameters inside our expression. They can be accessed via method parameters names.

```php
#[DbalWrite('INSERT INTO persons VALUES (:personId, :name, :roles)')]
#[DbalParameter(name: 'roles', expression: "name === 'Admin' ? ['ROLE_ADMIN'] : []", convertToMediaType: MediaType::APPLICATION_JSON)]
public function registerUsingMethodParameters(int $personId, string $name): void;
```

### Using Class Level Dbal Parameters

As we can use method level, we can also use class level Dbal Parameters. In case of Class level parameters, they will be applied to all the method within interface.

```php
#[DbalParameter(name: 'registeredAt', expression: "reference('clock').now()")]
class AdminAPI
{
    #[DbalWrite('INSERT INTO persons VALUES (:personId, :name, :registeredAt)')]
    public function registerAdmin(int $personId, string $name): void;
}
```

## Using Expression language in SQL

To make our SQLs more readable we can also use the expression language directly in SQLs.&#x20;

Suppose we Pagination class

```php
final readonly class Pagination
{
    public function __construct(public int $limit, public int $offset)
    {
    }
}
```

then we could use it like follows:

```php
interface PersonService
{
    #[DbalQuery('
            SELECT person_id, name FROM persons 
            LIMIT :(pagination.limit) OFFSET :(pagination.offset)'
    )]
    public function getNameListWithIgnoredParameters(
        Pagination $pagination
    ): array;
}
```

To enable expression for given parameter, we need to follow structure `:(expression)`, so to use limit property from Pagination class we will write `:(pagination.limit)`&#x20;
