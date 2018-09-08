# Dependency Injection

As stated in the *Initialize & Bootstrap* section, Event Machine pulls third-party dependencies from a [PSR-11 container](https://www.php-fig.org/psr/psr-11/).
It ships with its own 100 LoC container with the code name *Disco Light* (real class name is `ReflectionBasedContainer`).
The implementation of *Disco Light* is inspired by the design of [bitexpert/disco](https://github.com/bitExpert/disco)
but functionality is reduced to a bare minimum needed in a PHP 7.1+ environment. Hence, the code name ;)

## Required Services

The container needs to provide a list of services required by Event Machine. For each of them a constant is defined. See the table below:

Constant | Value | Service Description | Mandatory
---------|-------|---------------------|-------------
EventMachine::SERVICE_ID_EVENT_STORE | EventMachine.EventStore | prooph/event-store v7 event store | Yes
EventMachine::SERVICE_ID_SNAPSHOT_STORE | EventMachine.SnapshotStore | prooph/event-store v7 snapshot store | No
EventMachine::SERVICE_ID_COMMAND_BUS | EventMachine.CommandBus | prooph/service-bus v6 command bus | Yes
EventMachine::SERVICE_ID_EVENT_BUS | EventMachine.EventBus | prooph/service-bus v6 event bus | Yes
EventMachine::SERVICE_ID_QUERY_BUS | EventMachine.QueryBus | prooph/service-bus v6 query bus | Yes
EventMachine::SERVICE_ID_PROJECTION_MANAGER | EventMachine.ProjectionManager | prooph/event-store v7 projection manager | Yes
EventMachine::SERVICE_ID_DOCUMENT_STORE | EventMachine.DocumentStore | Store implementing DocumentStore interface | No
EventMachine::SERVICE_ID_ASYNC_EVENT_PRODUCER | EventMachine.AsyncEventProducer | prooph/service-bus v6 message producer | No
EventMachine::SERVICE_ID_MESSAGE_FACTORY | EventMachine.MessageFactory | prooph/common v4 message factory (default provided) | No
EventMachine::SERVICE_ID_JSON_SCHEMA_ASSERTION | EventMachine.JsonSchemaAssertion | Class implementing JsonSchemaAssertion interface (default provided) | No

## Default Services

Event Machine ships with a default implementation for the last two services, a dedicated `EventMachineContainer` that provides the services and a `ContainerChain`
to merge your container with the defaults:

```php
$myCustomContainer = include 'config/container.php';

$defaultsContainer = new \Prooph\EventMachine\Container\EventMachineContainer($eventMachine);

$psr11ContainerChain = new \Prooph\EventMachine\Container\ContainerChain($myCustomContainer, $defaultsContainer);

$eventMachine->initalize($psr11ContainerChain);
```

## Working With Disco Light

Remember the origin of Event Machine. It was designed as a workshop framework first. So one of the nice things about *Disco Light* is that dependencies are not wired together
by a magical component but instead by the developer. This way they can learn the different parts of the system and what configuration is needed to get everything
to work together. In the Event Machine Skeleton we've done that for you. A single class called the `ServiceFactory` is responsible for providing all services.

You can find the `ServiceFactory` of the skeleton [here](https://github.com/proophsoftware/event-machine-skeleton/blob/master/src/Service/ServiceFactory.php)

### Service Ids

Let's look at the method which provides the service `EventMachine.EventStore`:

```php
public function eventStore(): EventStore
{
    return $this->makeSingleton(EventStore::class, function () {
        $eventStore = new PostgresEventStore(
            $this->eventMachine()->messageFactory(),
            $this->pdoConnection(),
            $this->eventStorePersistenceStrategy()
        );
        return new TransactionalActionEventEmitterEventStore(
            $eventStore,
            new ProophActionEventEmitter(TransactionalActionEventEmitterEventStore::ALL_EVENTS)
        );
    });
}
```

A lot of stuff going on here, so we'll look at it step by step.

```php
public function eventStore(): EventStore
```

All `public` methods of the `ServiceFactory` are scanned by *Disco Light* (\Prooph\EventMachine\Container\ReflectionBasedContainer).
The **return type of the method** is used as as **service id**. This means that you can do the following to get the event store from the container:

```php
$eventStore = $container->get(EventStore::class);
```

### Singleton Service

In most cases we want to get the same instance of a service from the container no matter how often we request it. This is called a `Singleton`.
*Disco Light* is dead simple. It does not know anything about singletons. Instead we use a pattern called [memoization](https://en.wikipedia.org/wiki/Memoization)
to cache the instance of a service in memory and return it from cache on subsequent calls.

The `ServiceFactory` is a complete userland implementation. No interface needs to be implemented. To add memoization to your service factory you can use the provided
trait `\Prooph\EventMachine\Container\ServiceRegistry` like it is done in the skeleton service factory.

```php
final class ServiceFactory
{
    use ServiceRegistry;
```

Now you can store service instances in memory:

```php
public function eventStore(): EventStore
{
    return $this->makeSingleton(EventStore::class, function () {
        //...
    });
}
```

You might recognize that we use `EventStore::class` again as service id for the registry. The second argument of `makeSingleton` is a closure which acts
as a **factory function** for the service. When `EventStore::class` is not in the cache, the factory function is called otherwise the service is returned from the registry.

### Injecting Dependencies in a Service

Often one service depends on other services. The Postgres event store used in the skeleton for example requires a `MessageFactory` a `\PDO` connection and a `PersistenceStrategy`
and because all services are provided by the same `ServiceFactory` we can simply get those services by calling the appropriate methods.

{.alert .alert-light}
By default a closure is bound to its parent scope (the service factory instance in this case). Hence, insight the closure we have
access to all methods of the service factory no matter if they are declared public, protected or private.

```php
public function eventStore(): EventStore
{
    return $this->makeSingleton(EventStore::class, function () {
        $eventStore = new PostgresEventStore(
            $this->eventMachine()->messageFactory(),
            $this->pdoConnection(),
            $this->eventStorePersistenceStrategy()
        );
        return new TransactionalActionEventEmitterEventStore(
            $eventStore,
            new ProophActionEventEmitter(TransactionalActionEventEmitterEventStore::ALL_EVENTS)
        );
    });
}
```


The event store interface is service id and return type at the same time. Therefor, PHP's type system ensures at runtime that a valid event store is returned.
Internally, we built a Postgres event store and add prooph's plugin system (the TransactionalActionEventEmitterEventStore). If we want to switch the event store
we can return another implementation.

### Configuration

Another thing that is out of scope for *Disco Light* is application configuration. Remember, providing a working `ServiceFactory` is your task and if services
need configuration then pass it to the class. In the skeleton environmental variables are mapped to config params in
[config/autoload/global.php](https://github.com/proophsoftware/event-machine-skeleton/blob/master/config/autoload/global.php#L16).

The configuration array is then passed to the `ServiceFactory` in the constructor and wrapped with an `ArrayReader`:

```php
final class ServiceFactory
{
    use ServiceRegistry;

    //...

    public function __construct(array $appConfig)
    {
        $this->config = new ArrayReader($appConfig);
    }
```

This way we have access to the configuration when building our services. We can see this in action in the factory method of the `\PDO` connection:

```php
public function pdoConnection(): \PDO
{
    return $this->makeSingleton(\PDO::class, function () {
        $this->assertMandatoryConfigExists('pdo.dsn');
        $this->assertMandatoryConfigExists('pdo.user');
        $this->assertMandatoryConfigExists('pdo.pwd');
        return new \PDO(
            $this->config->stringValue('pdo.dsn'),
            $this->config->stringValue('pdo.user'),
            $this->config->stringValue('pdo.pwd')
        );
    });
}
```

`$this->assertMandatoryConfigExists(/*...*/)` is a helper function of the `ServiceFactory` marked as private. It is ignored by *Disco Light* but we can use
it within factory functions.

```php
private function assertMandatoryConfigExists(string $path): void
{
    if(null === $this->config->mixedValue($path)) {
        throw  new \RuntimeException("Missing application config for $path");
    }
}
```

{.alert .alert-info}
Again, this is all userland implementation. *Disco Light* does not care about it. If you don't like it to put all services
in a single class then use traits and only merge them in the `ServiceFactory`.
And if you don't like the approach at all, use another PSR-11 container! In any case you can learn from the skeleton service
factory how the mandatory services need to be wired together. Porting this knowledge to a container of your choice shouldn't be a problem.

### Service Alias

If you've read the explanations above carefully, you might have noticed a mismatch between the service id required by Event Machine and the
service id used in *Disco Light*. Event Machine requires the service id `EventMachine.EventStore`.
But we've learned that we get the event store by using the interface or class name as service id `$eventStore = $container->get(EventStore::class);`.

To solve the conflict we need a **service alias**. That said, the same service needs to be available in the container with two different ids.
We can do this by passing a service alias map to *Disco Light* aka `ReflectionBasedContainer`:

```php
$container = new \Prooph\EventMachine\Container\ReflectionBasedContainer(
    $serviceFactory,
    [
        \Prooph\EventMachine\EventMachine::SERVICE_ID_EVENT_STORE => \Prooph\EventStore\EventStore::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_PROJECTION_MANAGER => \Prooph\EventStore\Projection\ProjectionManager::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_COMMAND_BUS => \App\Infrastructure\ServiceBus\CommandBus::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_EVENT_BUS => \App\Infrastructure\ServiceBus\EventBus::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_QUERY_BUS => \App\Infrastructure\ServiceBus\QueryBus::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_DOCUMENT_STORE => \Prooph\EventMachine\Persistence\DocumentStore::class,
    ]
);
```

## Put it all together

Again, we can look at the skeleton for a working example. [config/container.php]:
- includes the application config
- passes it to a new instance of the `ServiceFactory`
- passes the `ServiceFactory` and service alias map to a new `ReflectionBasedContainer`
- hands over the container to the `ServiceFactory` (setter injection due to circular dependency)
- and finally returns the container

```php
<?php
declare(strict_types = 1);

$config = include 'config.php';

$serviceFactory = new \App\Service\ServiceFactory($config);

$container = new \Prooph\EventMachine\Container\ReflectionBasedContainer(
    $serviceFactory,
    [
        \Prooph\EventMachine\EventMachine::SERVICE_ID_EVENT_STORE => \Prooph\EventStore\EventStore::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_PROJECTION_MANAGER => \Prooph\EventStore\Projection\ProjectionManager::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_COMMAND_BUS => \App\Infrastructure\ServiceBus\CommandBus::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_EVENT_BUS => \App\Infrastructure\ServiceBus\EventBus::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_QUERY_BUS => \App\Infrastructure\ServiceBus\QueryBus::class,
        \Prooph\EventMachine\EventMachine::SERVICE_ID_DOCUMENT_STORE => \Prooph\EventMachine\Persistence\DocumentStore::class,
    ]
);
$serviceFactory->setContainer($container);

return $container;
```

*Note: The container is passed to the service factory because the factory needs to pass it to Event Machine as soon as Event Machine is requested for the first time.*
