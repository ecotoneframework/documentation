# Before we start tutorial

## Setup for tutorial

Depending on the preferences, we may choose tutorial version for&#x20;

* [`Symfony`](https://symfony.com/)&#x20;
* [`Laravel`](https://laravel.com/)
* [`Lite (No extra framework)`](../modules/ecotone-lite/)

1. Use [git](https://git-scm.com) to download **starting point** in order to start tutorial

{% tabs %}
{% tab title="Symfony" %}
```bash
git clone git@github.com:ecotoneframework/symfony-tutorial.git
# Go to symfony-tutorial catalog
```
{% endtab %}

{% tab title="Laravel" %}
```bash
git clone git@github.com:ecotoneframework/laravel-tutorial.git
# Go to laravel-tutorial catalog

# Normally you will use "php artisan" for running console commands
# To reduce number of difference during the tutorial
# "artisan" is changed to "bin/console"
```
{% endtab %}

{% tab title="Lite" %}
```bash
git clone git@github.com:ecotoneframework/lite-tutorial.git
# Go to lite-tutorial catalog
```
{% endtab %}
{% endtabs %}

2\. Run command line application to verify if everything is ready.&#x20;

{% hint style="success" %}
There are two options in which we run the tutorial:&#x20;

* _Local Environment_&#x20;
* _Docker (preferred)_
{% endhint %}

{% tabs %}
{% tab title="Docker" %}
```php
/** Ecotone Quickstart ships with docker-compose with preinstalled PHP 8.0 */
1. Run "docker-compose up -d"
2. Enter container "docker exec -it ecotone-quickstart /bin/bash"
3. Run starting command "composer install"
4. Run starting command "bin/console ecotone:quickstart"
5. You should see:
"Running example...
Hello World
Good job, scenario ran with success!"
```
{% endtab %}

{% tab title="Local Environment" %}
```php
/** You need to have atleast PHP 8.0 and Composer installed */
1. Run "composer install" 
2. Run starting command "bin/console ecotone:quickstart"
3. You should see:
"Running example...
Hello World
Good job, scenario ran with success!"
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
If you can see "Hello World", then we are ready to go. Time for Lesson 1!
{% endhint %}

{% content-ref url="php-messaging-architecture.md" %}
[php-messaging-architecture.md](php-messaging-architecture.md)
{% endcontent-ref %}
