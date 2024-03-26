# Registering new Module Package

If you want to provide new integration with external tooling, or you want to expose your set of features as separate package, then we will need to create new `Module Package`.

## Set up new Package

Ecotone dev repository comes with Module Template, you can find the template in `packages/_PackageTemplate`.\
\
Start by copying this catalog and replace placeholders with `_PackageTemplate` with your package name, for example `Dbal`.

## Registering new Module

Each Module needs to implements `AnnotationModule` and be annotated with `#[ModuleAnnotation]`.

You will find example `Module`, that you can adjust in `Ecotone\_PackageTemplate\Configuration\_PackageTemplate.`\
\
**create method:**\
Module is constructed from static `create` method.\
In this method you may use of `AnnotationFinder` to fetch for all the classes that you're interested in.

**canHandle method:**

This tell Ecotone what `extension objects` should be delivered to `prepare method.`\
Those objects are objects created in user land code by [Service Configuration](../service-application-configuration.md).\
\
**prepare method:**

This registers [Configuration](https://github.com/ecotoneframework/ecotone/blob/master/src/Messaging/Config/Configuration.php) for Ecotone, that can be executed on later stage.
