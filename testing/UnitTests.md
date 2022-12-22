---
layout: page
title: Unit Tests
parent: Testing
nav_order: 1
---

# How to write unit tests

## Test frameworks

In the SmartActors core [junit](http://junit.org/) and [mockito](http://site.mockito.org/) are used to test actors, plugins and other units. Mockito is utilized to create mock objects, for example for instantiating a wrapper from its interface.

There is no problem to unit test simple units which don't use IoC, because you just have a plain java services. But when you are to test an actor that retrieve something from IoC you have to initialize core components somehow. In this guide we show how to use IoC and scopes to do unit tests.

### Warning

You might think it's easier to use existing Plugins to register in the IoC objects but it's bad practice. When you run parallel testing you will face the problem of having unmanaged dependencies inside the IoC because of IoC being static. (TBDL)

## Register objects in IoC

The system has a special mechanism to split something called scopes to avoid parallel testing problem.

For example we have an actor requires SomeService.

```java
import info.smart_tools.smartactors.ioc.ioc.IOC;
import info.smart_tools.smartactors.ioc.key_tools.Keys;

public class SomeActor {
    public void handler(SomeActorWrapper wrapper) throws Exception {
        SomeService service = IOC.resolve(Keys.getKeyByName("SomeService"));
        service.doStuff();
        // some other code
    }
}
```

Yes, this service is registered in some plugin, but we would like to have a full pure mockable control, so just believe and use this pattern:

```java
import info.smart_tools.smartactors.base.strategy.apply_function_to_arguments.ApplyFunctionToArgumentsStrategy;
import info.smart_tools.smartactors.iobject.ds_object.DSObject;
import info.smart_tools.smartactors.iobject.field_name.FieldName;
import info.smart_tools.smartactors.iobject.ifield_name.IFieldName;
import info.smart_tools.smartactors.iobject.iobject.IObject;
import info.smart_tools.smartactors.ioc.ioc.IOC;
import info.smart_tools.smartactors.ioc.istrategy_container.IStrategyContainer;
import info.smart_tools.smartactors.ioc.key_tools.Keys;
import info.smart_tools.smartactors.ioc.resolve_by_name_ioc_with_lambda_strategy.ResolveByNameIocStrategy;
import info.smart_tools.smartactors.ioc.strategy_container.StrategyContainer;
import info.smart_tools.smartactors.ioc.string_ioc_key.Key;
import info.smart_tools.smartactors.scope.iscope.IScope;
import info.smart_tools.smartactors.scope.scope_provider.ScopeProvider;
import info.smart_tools.smartactors.task.interfaces.itask.ITask;
import org.junit.Before;
import org.junit.Test;


import static org.mockito.Mockito.*;

public class SomeActorTest {
    private actor = new SomeActor();

    @Before
    public void setUp() throws Exception {
        Object keyOfMainScope = ScopeProvider.createScope(null);
        IScope scope = ScopeProvider.getScope(keyOfMainScope);
        scope.setValue(IOC.getIocKey(), container);
        ScopeProvider.setCurrentScope(scope);
        // Something we HAVE TO REGISTER ALWAYS
        IOC.register(
                IOC.getKeyForKeyByNameStrategy(),
                new ResolveByNameIocStrategy(
                        (a) -> {
                            try {
                                return new Key((String) a[0]);
                            } catch (Exception e) {
                                throw new RuntimeException(e);
                            }
                        })
        );
        IOC.register(
                IOC.resolve(IOC.getKeyForKeyByNameStrategy(), IFieldName.class.getCanonicalName()),
                new ApplyFunctionToArgumentsStrategy(
                        (args) -> {
                            try {
                                return new FieldName((String) args[0]);
                            } catch (Exception e) {
                                throw new RuntimeException("exception", e);
                            }
                        }
                )
        );
        IOC.register(Keys.getKeyByName(IObject.class.getCanonicalName()),
                new ApplyFunctionToArgumentsStrategy(args -> {
                    if (args.length == 0) {
                        return new DSObject();
                    } else if (args.length == 1 && args[0] instanceof String) {
                        try {
                            return new DSObject((String) args[0]);
                        } catch (InvalidArgumentException e) {
                            throw new RuntimeException(e);
                        }
                    } else {
                        throw new RuntimeException("Invalid arguments for IObject creation.");
                    }
                })
        );

        // Now we can add our dependency
        SomeService someService = mock(SomeService.class);
        when(someService.doStuff()).thenReturn("good boy!");
        IOC.register(Keys.getKeyByName("SomeService"),
                new ApplyFunctionToArgumentsStrategy(args -> someService)
        );
    }

    public void Should_handle() throws Exception {
      SomeActorWrapper w = mock(SomeActorWrapper.class);
      actor.handler(wrapper);
      // some expectations....
    }
}
```

Use this code, but check imports, because it might be broken due to new versions.
