# Message Headers

Ecotone provides easy way to pass Message Headers with your Message and use it in your Message Handlers or [Interceptors](interceptors.md).\
In case of asynchronous scenarios, Message Headers will be automatically mapped and passed to your Message Broker.

## Passing headers to Bus:

Pass your metadata (headers), as second parameter.

```php
$this->commandBus->send(
   new CloseTicketCommand($request->get("ticketId")),
   ["executorUsername" => $security->getUser()->getUsername()]
);
```

Then you may access them directly in Message Handlers:

```php
#[CommandHandler]
public function closeTicket(
      CloseTicketCommand $command, 
      #[Header("executorUsername")] string $executor
) {
//        handle closing ticket
}  
```

If you have defined [Converter](../../messaging/conversion/method-invocation.md#default-converters) for given type, then you may type hint for the object and Ecotone will do the conversion:

```php
#[CommandHandler]
public function closeTicket(
      CloseTicketCommand $command, 
      #[Header("executorUsername")] Username $executor
) {
//        handle closing ticket
}
```

