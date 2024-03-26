---
description: Enterprise Integration Patterns PHP
---

# Message



![](../../.gitbook/assets/message.jpg)

A message is a generic wrapper for any PHP type (scalar, object, compound) with metadata used by the framework while handling that object.\
The payload can be of any type, and the headers (metadata) hold commonly required information such as ID, timestamp and framework specific information. Headers are also used for passing values to and from connected transports. \
Developers can also store any arbitrary key-value pairs in the headers, to pass needed meta information.

```php
interface Message
{
    public function getPayload();

    public function getHeaders() : MessageHeaders;
}
```

