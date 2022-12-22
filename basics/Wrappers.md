---
layout: default
title: Wrappers
parent: Basics
nav_order: 4
---

# Wrappers

The Actors in SmartActors are able to declare the data items they need from the system and they want to return to the system.
This is done with Wrappers.

## Handler and wrapper interface

The actor declares a handler method which receives the wrapper interface.

```java
public class HelloActor {
    public void hello(final GreetingMessage message) throws HelloActorException {
        // do something with GreetingMessage
    }
}
```

The interface defines the data pieces needed by the actor.

```java
public interface GreetingMessage {
    String getName() throws ReadValueException;
    void setGreeting(String greeting) throws ChangeValueException;
}
```

Getter is used to take data from the system IN to the actor.
So we called it IN-method.
It throws `ReadValueException` because sometimes the reading of the data may fail.

Setter is used to put data from the actor OUT to the system.
So we called it OUT-method.
It throws `ChangeValueException` because sometimes the writing of the data may fail.

Declaring such exceptions for getters and setters is the only requirement for the wrapper interface.

## Wrapper configuration

Which data to return by getters or receive in setters is defined by the wrapper configuration.
It's part of the message map definition.

For example:

```json
"wrapper": {
    "in_getName": "message/personName",
    "out_setGreeting": "response/greeting",
}
```

This was a short form of wrapper definition. Actually it's expanded to this.

```json
"wrapper": {
    "in_getName": [
        {
            "name": "wds_getter_strategy",
            "args": [ "message/personName" ]
        }
    ],
    "out_setGreeting": [[
        {
            "name": "wds_target_strategy",
            "args": [ "local/value", "response/greeting" ]
        }
    ]]
}
```

In-methods have a list of transformation strategies which are to extract data.
Out-methods have a list of a list of transformation strategies which are to set data.
The two nesting level for the setter is necessary because the same single value passed to the setter can be put to different set of system objects, to multiple destinations.

## Environment

Note the "message", "local" and "response" strings in the wrapper configuration above.
They are environment objects.

Each of them is represented as [IObject](./IObject): a set of named fields and values.
The values can be scalars, like strings and numbers, lists, nested IObjects or even plain Java objects.

To access the values for each environment object use a slash-separated path: `object_name/field_name`.
The path can go deeper to nested objects: `object_name/nested_object/nested_field`.

### Message

`message` is the current processing message represented as IObject.
It's the message received by the endpoint and processed by the previous actors in the message map.

Actors are able to modify fields of the message, add new fields, delete fields, etc...

Note one actor can use a setter of it's wrapper to set a value to the message, another actor can read this value using it's wrapper getter.
It's not necessary for both actors to negotiate the name of the message field they use, because all mapping, in both directions from actor to the message and from the message to the actor, are handled by the wrapper and are defined in the wrapper configuration.

### Response

`response` is the object (as IObject) which will be returned via endpoint as the response to the request.

Initially response is empty.
It is passed through all actors in the message map, each actor can add or modify fields in it.
Then the "respond" actor returns the response back to the client who sent the message to the endpoint.

### Local

There is only one value called `local/value`.
It's used to pass a value between transformation strategies.

For getters the initial `local/value` is `null`.
For setters the initial `local/value` is set to the object passed to the setter from the actor.
Then, for next strategies in the transformation chain (see below), the `local/value` is set to the result of execution of the previous strategy in the chain.

### Const

`const` is a special way to pass a constant string through the getter to your actor.

For example, `const/value` will pass the String "value" to the actor.

Think of such constants as a way to tune you actor in-place.
You define some getter in the wrapper to get some configuration parameter.
And you define the actual parameter value in the message map wrapper configuration.

### Context

`context` is used to keep some unserializable Java objects which are not part of request or response and are actual only during the current request processing.

The typical example of such object is HTTP request of the HTTP endpoint, it's available as `context/request`.
Also the context can be used to set HTTP headers and cookies to the HTTP response.

Note the context mostly contain data specified to the used endpoint and protocol.
Typical business logic actors should avoid to use `context`.
They should use `message` and `response` to interact with each other and the client.

## Transformation strategies

Methods of the wrapper are joined with environment objects through transformation strategies or rules.
Each strategy is just a strategy, registered in [IOC](./IOC), it takes some arguments and returns the result of computations.

### In/getter

Strategies for in-methods are to produce the value to be returned by the getter.
Typically they have one argument: the value from the environment to return.

The strategies can be combined to the chain — the array of strategies defined in the wrapper configuration.
In this case the next strategy in the chain can receive the result of the previous strategy as `local/value`.

The result of the last strategy in the chain is passed into the actor as the getter return value.

### Out/setter

Strategies for out-methods are to take the value passed to the setter and produce some changes in the environment.
Typically they have two arguments: the setter argument passed from the actor, available as `local/value`, and the environment field to set.

These strategies can be combined to the chains.
In this case the next strategy in the chain can receive the result of the previous strategy as `local/value`.

Because it can be necessary for one setter to modify multiple environment objects and fields it's possible to define multiple transformation chains for out-method.
This is why you need array in the array in the wrapper definition in the configuration.
The nested arrays are independent transformation chains which receive the same `local/value` to the first transformation strategy.
While the outer array is just a list of transformation chains.

The result of the last strategy in the chain for out-method is just ignored.

### Short syntax

If you don't need a special transformation rules but want just to get a value from the environment or set a value to the environment, you can use the short syntax.

```json
"wrapper": {
    "in_getName": "message/personName",
    "out_setGreeting": "response/greeting",
}
```

If you need to apply a custom transformation to environment values an receive the result from in-method, you have to define the transformation chain explicitly.
The result of the last transformation become the result of in-method without additional efforts.

```json
"wrapper": {
    "in_getName": [
        {
            "name": "concat_strategy",
            "args": [ "message/firstName", "const/ ", "message/lastName" ]
        }
    ]
}
```

If you need to apply a custom transformation to a value received as argument of out-method, you have to define the transformation chain explicitly.
A general transformation rule should not modify the environment directly, it should receive values as arguments and return the transformation result as `local/value`.
In this case it's necessary to target the result of the last transformation to the specific environment field.
There is a special short syntax for it.

```json
"wrapper": {
    "out_setName": [[
        {
            "name": "split_strategy",
            "args": [ "local/value", "const/ " ]
        },
        {
            "name": "target",
            "args": [ "response/namesList" ]
        }
    ]]
}
```

The short name "target" is expanded into built-in "wds_target_strategy".

```json
{
    "name": "wds_target_strategy",
    "args": [ "local/value", "response/namesList" ]
}
```

## How to write your own transformation strategy

The transformation strategy or rule is a class implementing [IStrategy](../apidocs/info/smart_tools/smartactors/core/istrategy/IStrategy.html).
It takes some Object arguments and returns some value.
You can define it as an anonymous class.

```java
IStrategy strategy = new IStrategy() {
    
    @Override
    public <T> T resolve(final Object... args) throws StrategyException {
        String result = Arrays.stream(args).map(String::valueOf).collect(Collectors.joining());
        return (T) result;
    }
};
```

Then you need to register it in IOC.
Because in this case IOC should resolve the strategy, the key is constructed from the interface canonical name.

```java
IKey key = Keys.getKeyByName(IStrategy.class.getCanonicalName());
```

And you need the unique name for your strategy.

```java
String name = "concat_strategy";
```

Finally you should take strategy which is able to resolve strategy and pass it the name and the implementation of your strategy.
The strategy for strategies is already registered in IOC by `PluginWDSObject`.

```java
IOC.resolve(key, name, strategy);
```

After this you can mention you strategy name in the wrapper configuration.

```json
"in_getName": [
    {
        "name": "concat_strategy",
        "args": [ "message/firstName", "const/ ", "message/lastName" ]
    }
]
```

## Under the hood

### Config normalization

A special implementation of IObject, [ConfigurationObject](../apidocs/info/smart_tools/smartactors/core/configuration_object/ConfigurationObject.html), is used to parse config file and wrapper definitions in it.
This implementation expands short definitions in the wrapper into long definitions with lists of strategies.

```java
IKey configObjectKey = Keys.getKeyByName("configuration object");
ConfigurationObject config = IOC.resolve(configObjectKey,
    "{" +
    "\"in_getName\": \"message/personName\"," +
    "\"out_setGreeting\": \"response/greeting\"" +
    "}"
);
```

This code produces the IObject equivalent to the expanded version of the config mentioned above.

The strategies to use ConfigurationObject are registered in IOC by [InitializeConfigurationObjectStrategies](../apidocs/info/smart_tools/smartactors/plugin/configuration_object/InitializeConfigurationObjectStrategies.html) plugin.

### WDSObject

[WDSObject](../apidocs/info/smart_tools/smartactors/core/wds_object/WDSObject.html) is built over the configuration object.

```java
IObject config = IOC.resolve(configObjectKey,
    "{" +
    "\"in_getName\": \"message/personName\"," +
    "\"out_setGreeting\": \"response/greeting\"" +
    "}"
);
IKey wdsObjectKey = Keys.getKeyByName(WDSObject.class.getCanonicalName());        
WDSObject wdsObject = IOC.resolve(wdsObjectKey, config);
```

It's created for each `"wrapper"` section in the config.

For each `"in_"` or `"out_"` definition WDSObject creates the fields inside itself with the same names.
Each such field incapsulates the transformation strategies defined in the config.

WDSObject is initialized by the environment object when it's necessary to process the message.

```java
IObject environment = IOC.resolve(iObjectKey,
    "{" +
    "\"message\": { \"personName\": \"Ivan\" }," +
    "\"response\": {}" +
    "}"
);
wdsObject.init(environment);
```

The environment contains the message, context, response, etc... fields to process.

Interactions with `"in_"`/`"out_"` fields of WDSObject are transformed to interactions with the environment passed to `init()` method earlier.

The strategies to use WDSObject are registered by [PluginWDSObject](../apidocs/info/smart_tools/smartactors/plugin/wds_object/PluginWDSObject.html) plugin.

### WrapperGenerator

[WrapperGenerator](../apidocs/info/smart_tools/smartactors/core/wrapper_generator/WrapperGenerator.html) generates a class in runtime which implements IObject, IObjectWrapper and the wrapper interface — interface of the parameter of the actor's handler.

```java
IKey iWrapperGeneratorKey = Keys.getKeyByName(IWrapperGenerator.class.getCanonicalName());
IWrapperGenerator generator = IOC.resolve(iWrapperGeneratorKey);
GreetingMessage message = generator.generate(GreetingMessage.class);
```

Instance of this class is initialized by WDSObject created on the previous step.

```java
IObjectWrapper wrapper = (IObjectWrapper) message;
wrapper.init(wdsObject);
```

Then the instance is passed to the actor's handler.
Each call to it's methods is just translated into access to WDSObject fields causing the strategies to be applied to data and the result to affect the environment.

```java
assertEquals("Ivan", message.getName());

message.setGreeting("Hello");

IField responseField = IOC.resolve(iFieldKey, "response");
IObject response = responseField.in(environment);
IField greetingField = IOC.resolve(iFieldKey, "greeting");
assertEquals("Hello", greetingField.in(response));
```

The strategy to use WrapperGenerator are registered by [RegisterWrapperGenerator](../apidocs/info/smart_tools/smartactors/plugin/wrapper_generator/RegisterWrapperGenerator.html) plugin.

![Relations between objects in Wrapper](http://www.plantuml.com/plantuml/img/RL0n3i8m3Dpz2d-WVG52Gh0m80O6HcGYjmfgxIf9WO77KweG6f4bbvplpdQew_HnwUtJIWjW9R1ho33kZzSRc_3Fd1qD0xj5uS3UKyi0fYFlyIk8hzqf9kaCo7AtJgLd2L6oLMbiEpeALYMAWKnGXvlF2J03LkkK7H6rMZH8juckYga_nPIr70JY3hXwrNNgaczShELasP46p3s9plYw_1sPDPly3G00)

## Code

* [Some tests](../core.examples/xref-test/info/smart_tools/smartactors/core/examples/WrapperExample.html) demonstrating how config normalization, WDS object and WrapperGenerator works.
* [Plugin](../xref/info/smart_tools/smartactors/core/examples/wrapper/ConcatSplitRulesPlugin.html) which defines two sample transformation rules.
