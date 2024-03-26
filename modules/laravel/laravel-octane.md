# Laravel Octane

[Laravel Octane](https://github.com/laravel/octane) supercharges your application's performance by serving your application using high-powered application servers, including [Open Swoole](https://openswoole.com/), [Swoole](https://github.com/swoole/swoole-src), and [RoadRunner](https://roadrunner.dev/).

In order for frameworks or libraries to be used with Octane they must be stateless, otherwise it will produce memory leaks or may affect the results of next requests.&#x20;

{% hint style="success" %}
Ecotone is safe to be used with Laravel Octane as it works in stateless way, this way memory and consecutive requests are not affected.
{% endhint %}
