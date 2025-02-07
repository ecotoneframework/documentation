# Intercepting Asynchronous Endpoints

Read [previous section](interceptors/) to find out more about Interceptors.

## Intercepting Asynchronous Endpoints

We may aswell intercept Asynchronous Endpoints pretty easily. We do it by using pointing to **AsynchronousRunningEndpoint** class.

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
