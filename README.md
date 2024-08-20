<p align="left"><img src="https://raw.githubusercontent.com/laravel/octane/2.x/art/logo.svg" alt="Logo Laravel Octane"></p>

## 🚀 Laravel Octane Best Practices

A compiled list of [Laravel Octane](https://laravel.com/docs/11.x/octane) best practices for your team to follow.

If you use Octane, remember, you're in the domain of stateful applications.
This means that the RAM it uses is no longer reset between requests but lives inside on-demand bootstrapped processes (workers).
While Octane can significantly enhance speed, it also introduces specific constraints and potential challenges, such as managing application state and ensuring memory leaks are controlled.
This repository accumulates common practices to avoid issues Octane can potentially introduce to your application if you're not aware of them.

## ❌ Avoid static variables and properties
Don't use properties or variables with the `static` keyword:
```php
static $propertyOrVariable = 0;
```

If you increase the integer in this property, the value won't be reset after the request is finished, but instead when the worker finishes its lifecycle.
Following users might see the changes that were applied to the property by the previous visitor. Furthermore, entities like this can lead to memory leaks if you do it like this:
```php
Service::$data[] = $data;
```

Not quite the thing we expect to achieve, right? Keep that in mind!

## ⚠️ Use singletons carefully
Prevent unintended data sharing between requests when using singletons.
Generally, register singletons only for things that are fully stateless.
"Stateless" means that the object does not maintain any internal state between interactions with the object.

The example from [Laravel](https://laravel.com/docs/11.x/octane#configuration-repository-injection) documentation.

Problem:
```php
$this->app->singleton(Service::class, function (Application $app) {
    return new Service($app);
});
```

In this example, if the Service instance is resolved during the application boot process, the container will be injected into the service and that same container instance will be held by the Service instance on subsequent requests.
This may not be a problem for your particular application.
However, it can lead to the container unexpectedly missing bindings that were added later in the boot cycle or by a subsequent request.

Solution:
```php
$this->app->singleton(Service::class, function () {
    return new Service(fn () => Container::getInstance());
});
```

You can also avoid using singletons or use [scoped singletons](https://laravel.com/docs/11.x/container#binding-scoped) instead.

## 🗄️ Database connections are persistent by default
When using Octane, database connections are persistent by default. This is OK for most apps and improves the performance of your database interactions by keeping the connection.
But in some cases, you might want to ensure that database connections do not persist across requests due to the specifics of your application.

To achieve this, uncomment [this line](https://github.com/laravel/octane/blob/2.x/config/octane.php#L108) in your `octane.php` config:
```php
DisconnectFromDatabases::class,
```

## ⚙️ Choose the right driver
Carefully review and choose [the driver](https://laravel.com/docs/11.x/octane#server-prerequisites) that fits your application requirements. Make the decision wisely.

Side note: While RoadRunner is easier to set up and provides lots of integrations, Swoole has additional features that Octane can utilize natively, like [concurrent tasks](https://laravel.com/docs/11.x/octane#concurrent-tasks) or [Octane Cache](https://laravel.com/docs/11.x/octane#the-octane-cache).

## 📊 Optimize worker counts
By default, Octane will start an application request worker for each CPU core provided by your machine.
Always increase these numbers for performance, but look not only at the amount of vCPU your machine has but also at the `memory_limit` of your processes.
Each worker is a CLI process, similar to the processes spawned from Artisan commands like `php artisan queue:work` if you're familiar with these.
Adjust, monitor, and experiment with these values to achieve the best performance ratio possible on the server resources currently available.

## 🔄 Reset workers after X requests
To avoid memory leaks, you can introduce the safety technique on the worker level.

By setting the `--max-requests=500` on the `php artisan octane:start` command you can tell the worker to gracefully reload itself after it handles 500 requests.
This way, even if a memory leak appears in the application, the memory will be freed automatically after the specified threshold of requests.

## 🔍 Monitor queuing
In Octane (technically in the underlying driver), each request is being "queued" to your process if the process is currently busy.
Some drivers, like Swoole (or OpenSwoole), allow you to track the "queued" requests to your workers. This can help in the understanding of your current load.

Like this:
```php
app(\Swoole\Http\Server::class)->stats(1);
```

```
connection_num	30
abort_count	0
accept_count	34
close_count	26
worker_num	8
task_worker_num	8
user_worker_num	0
idle_worker_num	0
...
```

These stats say that your worker number is too low to handle the incoming connections quickly since you have only `8` workers to handle 30 incoming connections, so you might consider increasing the worker count.

## ⚔️ Avoid race conditions
Imagine the situation when you have a heavy database query to calculate the number of users online.
You can cache this operation to reduce the load on your database. However, if you run your application with Octane and spawn a lot of workers, the race condition might appear when acquiring the cache, and the query will be executed multiple times depending on the incoming traffic and the amount of workers.

To prevent this, you should use [atomic locks](https://laravel.com/docs/11.x/cache#atomic-locks).

---

Thank you for reading! I hope it saved your team time and energy.\
If you see anything we can add or improve in this article, [send a PR](https://github.com/michael-rubel/laravel-octane-best-practices/compare).
