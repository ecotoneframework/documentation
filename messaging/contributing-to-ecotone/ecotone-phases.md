# Ecotone Phases

## Phases

To increase application speed **Ecotone works in three phases**.

## **Compile Phase:**

In this Phase, Ecotone is looking for all the configuration. This is optimized to look only for concrete Ecotone classes and scan our application level catalog (Depending on the configuration).

After all configuration is collected, Ecotone executes Modules in order to prepare them.\
Each Module produces it's own set of configuration, like Command Handlers, Event Handlers, Interceptors etc.  After this all the configuration is registered in Messaging system and we are ready to move to the next phase.

## **Dumping Phase:**

After all configuration is collected, Ecotone will dump the configuration, so it can be reused between requests. This depending on the integration, may be dumped directly to our Depedency Container or as a separate configuration file.&#x20;

The configuration now is ready to be re-used.&#x20;

## Execution Phase:

Execution Phase is executing Messaging Flow. This happen when we actually trigger Command or Query Bus on Run Time for example.\
In this phase there is no need for collecting or dumping the configuration anymore, as that was already done. Therefore Ecotone load already prepared configuration and executes.&#x20;

The big advantage of Ecotone is that during Execution Phase, it only executes and load the configuration for the flow that is being executed. This way the speed of execution can be preserved even for big projects.
