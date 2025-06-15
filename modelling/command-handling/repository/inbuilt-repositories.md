# Inbuilt Repositories

Ecotone comes with inbuilt repositories, so we don't need to configure Repositories ourselves. It often happen that those are similar between projects, therefore it may be that there is no need to roll out your own.

## Inbuilt Repositories

Ecotone provides inbuilt repositories to get you started quicker. This way you can enable given repository and start implementing higher level code without worrying about infrastructure part.

### Doctrine ORM Support

This provides integration with [Doctrine ORM](https://www.doctrine-project.org/projects/orm.html). To enable it read more in [Symfony Module Section](../../../modules/symfony/doctrine-orm.md).

### Laravel Eloquent Support

This provides integration with [Eloquent ORM](https://laravel.com/docs/5.0/eloquent/). Eloquent support is available out of the box after installing [Laravel module](../../../modules/laravel/laravel-ddd-cqrs-event-sourcing.md).

### Document Store Repository

This provides integration [Document Store](../../../messaging/document-store.md) using relational databases. It will serialize your aggregate to json and deserialize on load using [Converters](../../../messaging/conversion/conversion/).\
To enable it read in [Dbal Module Section](../../../modules/dbal-support.md#document-store).

### Event Sourcing Repository

Ecotone provides inbuilt Event Sourcing Repository, which will set up Event Store and Event Streams. To enable it read [Event Sourcing Section](../../event-sourcing/).
