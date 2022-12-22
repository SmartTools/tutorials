---
layout: default
title: Scopes
parent: Basics
nav_order: 2
---

# Scopes

## Overview

Scope is a key-value storage, where system [service locators]((https://en.wikipedia.org/wiki/Service_locator_pattern)) like [IoC](IOCExample) are able to store their internal data.

### Requirement for the keys

Because the data is internal, it's better to use a random [GUID](https://en.wikipedia.org/wiki/Globally_unique_identifier) as a key to private data. This key is known only to the users of the data, other users cannot guess the key.

```java
Object scopeKey = ScopeProvider.createScope(null);
IScope scope = ScopeProvider.getScope(scopeKey);
Object key = new Key(java.util.UUID.randomUUID().toString());
Object value = new Object();
scope.setValue(key, value);
// ...
value = scope.getValue(key)
```

### Why you need scopes        

Service locators provide the globally available API, and the scopes give ability to separate the data of service locators in different contexts. For example, you can define totally independent scope for testing environment, so your tests will take the IoC data totally independent on the IoC of the main application.

The service locator should always take it's data from the _current scope_. But the current scope can be defined and changed externally.

Also scopes can be nested into each other. This allows to _override_ data. If some key is not defined in the current scope, it may be looked in the parent scope. As the opposite you can redefine the value in the current scope, and it doesn't affect the users of the parent scope.

## Nested scopes

For example, let define a main scope. It's has no parents, so `null` is passed to `createScope`.

```java
Object mainScopeKey = ScopeProvider.createScope(null);
IScope mainScope = ScopeProvider.getScope(mainScopeKey);
ScopeProvider.setCurrentScope(mainScope);
```

You can put a value to it. It'll be a main value.

```java
Object mainValue = new Object();
ScopeProvider.getCurrentScope().setValue(key, mainValue);
assertSame(mainValue, ScopeProvider.getCurrentScope().getValue(key));
```

Now you define a nested scope and make it the current. Pass a main scope as a parent to `createScope`.

```java
Object nestedScopeKey = ScopeProvider.createScope(mainScope);
IScope nestedScope = ScopeProvider.getScope(nestedScopeKey);
ScopeProvider.setCurrentScope(nestedScope);
```

When you read by the same key, you get the value from the main scope.

```java
assertSame(mainValue, ScopeProvider.getCurrentScope().getValue(key));
```

You can put a new value to the nested scope. Then you'll read the updated value.

```java
Object nestedValue = new Object();
ScopeProvider.getCurrentScope().setValue(key, nestedValue);
assertSame(nestedValue, ScopeProvider.getCurrentScope().getValue(key));
```

However, when you return back to the main scope, you'll read the original value.

```java
ScopeProvider.setCurrentScope(mainScope);
assertSame(mainValue, ScopeProvider.getCurrentScope().getValue(key));
```

### Why you need nested scopes

Imagine you have some system code, which should use a system scope for it's needs. However, you have to call some worker code which is not fully trusted. You want to give the worker all resources you system code have. But you want to isolate worker so it doesn't corrupt system data.

You define two scopes: system and worker. And worker is nested into system.

```java
Object systemScopeKey = ScopeProvider.createScope(null);
systemScope = ScopeProvider.getScope(systemScopeKey);

Object workerScopeKey = ScopeProvider.createScope(systemScope);
workerScope = ScopeProvider.getScope(workerScopeKey);

ScopeProvider.setCurrentScope(systemScope);
```

#### Call untrusted code

You run a system code and have to call some untrusted worker code. You change scope to worker's before call and return it back to system after call.

```java
ScopeProvider.setCurrentScope(workerScope);
workerCall();
ScopeProvider.setCurrentScope(systemScope);
```

Now you sure the worker cannot damage your system data if the worker cannot change the current scope. The worker must not receive the reference to the system scope, so it cannot switch to it.

#### Do a system call

Otherwise, when a worker code needs to do a system call.

```java
assertSame(workerScope, ScopeProvider.getCurrentScope());
systemCall();
assertSame(workerScope, ScopeProvider.getCurrentScope());
```

The system call should remember the scope it was called from and temporary switch to the system scope. And before return back to the worker, restore the initial scope.

```java
private void systemCall() throws ScopeProviderException {
    IScope returnScope = ScopeProvider.getCurrentScope();
    ScopeProvider.setCurrentScope(systemScope);
    // do something system
    ScopeProvider.setCurrentScope(returnScope);
}
```

So the system code can always run in system scope whatever scope it's called from. The system code should keep a reference to the system scope to do this.

## Sample IoC

### IoC requirements

Let's create the simple IoC which takes care about scopes. The IoC will be simple, it just always returns the objects put into it by the key.

```java
IKey key = new Key("sample");
SampleClass main = new SampleClass("main");
SimpleIOC.register(key, main);
assertEquals(main, SimpleIOC.resolve(key));
```

The IoC must return the parent scope object from the child scope.

```java
ScopeProvider.setCurrentScope(workerScope);
assertEquals(main, SimpleIOC.resolve(key));
```

But it should allow to override the object.

```java
SampleClass worker = new SampleClass("worker");
SimpleIOC.register(key, worker);
assertEquals(worker, SimpleIOC.resolve(key));
```

Also it should return to the main scope object when the scope is switched back.

```java
ScopeProvider.setCurrentScope(systemScope);
assertEquals(main, SimpleIOC.resolve(key));
```

### IoC storage

IoC must have a storage, and keep this storage in the scope. The storage is a separate class.

The storage can be based on a simple [Map](http://docs.oracle.com/javase/8/docs/api/java/util/Map.html).

```java
private final Map<K, V> storage = new ConcurrentHashMap<>();
```

So, to save the value, just put it to the map.

```java
public void put(final K key, final V value) {
    storage.put(key, value);
}
```

However, the storage must do a recursive search: if the key is not found in the current scope, ask the parent scope.

```java
private final RecursiveContainer<K, V> parent;

public V get(final K key) {
    V result = null;
    result = storage.get(key);
    if (result == null && parent != null) {
        result = parent.get(key);
    }
    return result;
}
```

The parent container is provided in the constructor.

```java
public RecursiveContainer(final RecursiveContainer<K, V> parent) {
    this.parent = parent;
}
```

### IoC responsibility    

Our IoC needs a key to save the storage in the scope.

```java
STORAGE_KEY = new Key(java.util.UUID.randomUUID().toString());
```

And it needs a code which guarantees the storage is correctly created in any new scope appeared in the system. Before adding a new storage to the scope we should ask the new scope for the parent storage (from the parent scope) to correctly link them.

```java
ScopeProvider.subscribeOnCreationNewScope(
    scope -> {
        try {
            RecursiveContainer<IKey, Object> parentStorage = null;
            try {
                parentStorage = (RecursiveContainer<IKey, Object>) scope.getValue(STORAGE_KEY);
            } catch (ScopeException e) {
                // parent storage does not exists, create a new with null parent
            }
            
            scope.setValue(STORAGE_KEY, new MyRecursiveContainer<>(parentStorage));
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
);
```

Then to register an object in IoC we just take the storage from the current scope and put a value into it.

```java
public static void register(final IKey key, final Object value) throws SimpleIOCException {
    try {
        RecursiveContainer<IKey, Object> storage = (RecursiveContainer<IKey, Object>)
            ScopeProvider.getCurrentScope().getValue(STORAGE_KEY);
        storage.put(key, value);
    } catch (Throwable e) {
        throw new SimpleIOCException(e);
    }
}
```

To resolve the key we take the storage from the current scope again and get the value. The recursive search is done by the storage.

```java
public static <T> T resolve(final IKey<T> key) throws SimpleIOCException {
    try {
        RecursiveContainer<IKey, Object> storage = (RecursiveContainer<IKey, Object>)
            ScopeProvider.getCurrentScope().getValue(STORAGE_KEY);
        return (T) storage.get(key);
    } catch (Throwable e) {
        throw new SimpleIOCException(e);
    }
}
```

### IoC objects

So, the scopes are key-value storages with hierarchical structure which allows to search values from child to parent.

But IoC internally is also the key-value storage which holds objects or strategies to resolve objects. And this storage should also be hierarchical with ability to search values from child to parent. The hierarchy of IoC storage follows the hierarchy of scopes.

So, the IoC as a service locator just takes the storage from the current scope and asks it for data. Because you can define different scopes for different situations you can get different (and independent) data from IoC when necessary.

![IoC in Scopes](http://www.plantuml.com/plantuml/img/XP2n3i8W48PtdkBIgHruWQRfqib94-FM1V5IgmW4Q1FVNi2ge4sS2Dnt_tq7uquPE5WqX71rqqgYTB1H7JIDVvn7ZY0KPvvgsnJPCUEFFLSQFNh5EvsPcD137wOxZ-AqXlpc-7ms_4jQXKYG1qhRQ5s3mM6q3arPzTAWLMB6iY8a50DKyCc4YRsqGQn89MiOq9LL3SiaGf9YRDRHjaYtpo15CrMI_fAlPSj-dBBjQj1JbDWUByYzv-BqlBwcBV2Qr3ldg_41)

## Code

The sources of this tutorial:

* [SimpleIOC implementation](../xref/info/smart_tools/smartactors/core/examples/scope/package-frame.html)
* [Tests](../core.examples/xref-test/info/smart_tools/smartactors/core/examples/ScopeExample.html)
