---
description: Intercepting asynchronous message endpoints in Ecotone PHP
---

# Intercepting Asynchronous Endpoints

You want a transaction wrapper, but only for handlers running asynchronously — synchronous Command handlers already get one from your Command Bus. Or you want to log async-only metrics, swap connections per-tenant, or apply rate-limiting to background work without touching synchronous flows. Pointing the interceptor at the `AsynchronousRunningEndpoint` class targets only the async path.

Read [previous section](interceptors/) for an introduction to Interceptors.

## Intercepting Asynchronous Endpoints

You target asynchronous endpoints by using **AsynchronousRunningEndpoint** as the pointcut.

```php
class TransactionInterceptor
{
    #[Around(pointcut: AsynchronousRunningEndpoint::class)]
    public function transactional(MethodInvocation $methodInvocation)
    {
        $this->connection->beginTransaction();
        try {
            $result = $methodInvocation->proceed();

            $this->connection->commit();
        }catch (\Throwable $exception) {
            $this->connection->rollBack();

            throw $exception;
        }

        return $result;
    }
}
```

### Inject Message's payload

As part of around intercepting, if we need Message Payload to make the decision we can simply inject that into our interceptor:

```php
#[Around(pointcut: AsynchronousRunningEndpoint::class)]
public function transactional(
    MethodInvocation $methodInvocation,
    #[Payload] string $command
)
```

### Inject Message Headers

We can also inject Message Headers into our interceptor. We could for example inject Message Consumer name in order to decide whatever to start the transaction or not:

```php
#[Around(pointcut: AsynchronousRunningEndpoint::class)]
public function transactional(
    MethodInvocation $methodInvocation,
    #[Header('polledChannelName')] string $consumerName
)
```
