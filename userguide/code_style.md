---
layout: default
title: Code Style
parent: User Guide
nav_order: 1
---

# Code style and guidelines
## Overview
The designation of this document is to describe recommended code style used in SmartActors projects. Although some recommendations in this document can be seen in other User Guide documents, this document serves as a single point of reference for code style.

## Java
To enforce code style in Java project, we recommend to use CheckStyle plugin in IntelliJ IDEA with [this configuration file](checkstyle.xml). Please note that this config was tested with CheckStyle version `8.18`, it may not work to full extend in latest versions.

### General code style
* Lenght of the code line is limited with 120 characters
    * If code line is exceeding this limit, then it should be split
* Block indentaion should be 4 spaces
* Interface names are written in `UpperCamelCase` style and contain `I` prefix in it.
* Exceptions must be caught and wrapped by method's specific exception
* Exception message must describe the reason the exception was thrown

**Example**:
```java
try {
    IObject document = Optional.ofNullable(message.getDocument())
        .orElseThrow(() -> new InsertDocumentException("Document cannot be null"));
} catch (ReadValueException e) {
    throw new InsertDocumentException("Unable to get document from the message", e);
} catch (RuntimeException e) {
    throw new InsertDocumentException("Runtime exception was caught", e);
}
```

### Actors
* Actor is represented as a single class file
* Only one actor may be present in the source file
* Actor must be `public`
* Actor naming guidelines:
    * Actor name is written in English, using no more than 4-6 words with `UpperCamelCase` style
    * Actor name does not contain word "Actor"
    * Actor name represents actor's purpose
* Actor should not contain overloaded methods
* Actor must be documented with JavaDoc

**Example**: `StatusCodeSetter.java` with actor `StatusCodeSetter`

#### Handlers
* Handler must be of type `public void` and receive only one `final` parameter - message wrapper
* Handler naming guidelines:
    * Handler name represents the action it supposed to do
    * Handler name is written in English with `lowerCamelCase` style
    * Handler represents single action of the actor
* Handler must be documented with JavaDoc

**Example**: `public void getDocument(final GetDocumentMessage message)`

#### Exceptions
* Exception naming guidelines:
    * Exception name represents the reason exception is thrown
    * Exception name is written in English with `UpperCamelCase` style and contains word `Exception` in it
* One handler may throw several exception if necessary, although it is recommended to stay with one exception
* Exception must be documented with JavaDoc

**Example**: `EmptyUserListException.java`

#### Wrappers
* Wrapper is represented as a single interface file
* Wrapper must be `public`
* For each handler there may be only one wrapper
* Wrapper contains only getters and setters for message fields
* Getter should throw `ReadValueException`
* Getter should not contain any argument
* Getter cannot use primitive types (e.g. `int`, `float`, `double`, etc.), only their class wrappers (e.g. `Integer`, `Double`, etc.)
* Setter should throw `ChangeValueException`
* Setter must return `void` type
* Wrapper naming guidelines:
    * Wrapper name represent the handler using it
    * Wrapper name is written in English with `UpperCamelCase` style and contain `Message` word in it
* Getter and setter naming guidelines:
    * Getter name represent what kind of entity it supposed to get from the message
    * Getter name is written in English with `lowerCamelCase` style and contain `get` word in the beginning
    * Setter name represent what kind of entity it supposed to set to the message
    * Setter name is written in English with `lowerCamelCase` style and contain `set` word in the beginning
* Wrapper itself must be documented with JavaDoc
* Each getter and setter must be documented with JavaDoc

**Example**:
```java
public interface SendEmailMessage {

    IObject getMessage() throws ReadValueException;

    void setResult(String result) throws ChangeValueException;
}
```

**Note**: in this example JavaDoc documentation is skipped, but it must be present in the actual code.

##### Configuration wrapper
These are the wrappers used in stateful actor's constructors. They're mostly similar to the handler wrapper with some changes in naming:

* Configuration wrapper name represents actor it supposed to help configure
* Wrapper name is written in English with `UpperCamelCase` style and contain `Config` word in it

**Example**: `EmailSenderConfig` wrapper for `EmailSender` actor.

#### Internal state
* Stateful actor's state is represented with private variables
* To initialize private variables, constructor with configuration wrapper is used
* Private variables are written in English with `lowerCamelCase` style
    * If private variable is a field name (i.e. it's of type `IFieldName`), then `FN` postfix must be appended
* `static final` variables should be avoided

**Example**:
```java
private IFieldName roundsFN;
private Integer hashRounds;
```

### Strategies and rules
* Transformation rule (or simply rule) or strategy must implement interface `IStrategy`
* Rule naming guidelines:
    * Rule must represent types it converts
    * Rule class name is written in English using `UpperCamelCase` with `Rule` word added at the end
* Strategy naming guidelines
    * Strategy name represents what it supposed to perform
    * Strategy class name is written in English using `UpperCamelCase` with `Strategy` word added at the end
* Rules and strategies cannot have internal state, treat them as a pure functions

**Example**:
```java
public StringToIntegerRule extends IStrategy {}
public CreateObjectStrategy extends IStrategy {}
```

### Plugins
* Plugin is represented as a single class file
* Only one plugin may be present in the source file
* Plugin must be `public` and inherit `BootstrapPlugin` abstract class
* Plugin may register only one actor or strategy in IOC at the time
    * This method should be marked with `@Item` annotation
* Plugin must contain method to unregister actor or strategy from IOC
    * This method should be marked with `@ItemRevert` annotation
* Plugin naming guidelines:
    * Plugin name represent actor or strategy it supposed to register
    * Plugin name is written in English with `UpperCamelCase` style with `Plugin` word in it.
* Plugin and it's methods must be documented with JavaDoc
* Plugin's JavaDoc must contain info on how to resolve actor or strategy from IOC

**Example**:
```java
public class StatusCodeSetterPlugin extends BootstrapPlugin {

    public StatusCodeSetterPlugin(final IBoostrap bootstrap) {
        super(bootstrap);
    }

    @Item("status-code-setter-plugin")
    public void init() {
        IOC.register(actorKey, actorStrategy);
    }

    @ItemRevert("status-code-setter-plugin")
    public void unregister() {
        IOC.unregister(actorKey);
    }
}
```

**Note**: in this example JavaDoc are skipped and process of registering actor in IOC is simplified.

## IOC and features
### IOC dependencies
* Dependency name is written in English using no more than 4-6 words separated by spaces
* Class name or interface name cannot be used as a dependency name
* `#` is used to set namespace in IOC
    * Namespace naming guidelines is the same as for all dependencies
* Dependency name should reflect type of the dependency registered in the IOC

**Example**:
```java
"get document executor"
"token manager#auth token manager"
"create object strategy"
```

### Features
* Feature may contain chains, actors or both
* Feature may contain only one external chain
* Feature naming guidelines:
    * Feature name is representative of it's content - either it's a bundle of actors or it contains external chain
    * Feature name is written in English using `lisp-case`
    * Feature name shouldn't contain word `feature` in it

**Example**:
* `database-search-criteria-builder` - feature with actors helping to build search criteria
* `sign-up` - feature with chain for sign up

#### Chains and actors
* Structure of configuration file is following:

```json
{
  "featureName": "feature",
  "afterFeatures": [],
  "objects": [],
  "maps": [],
  "onFeatureLoading": []
}
```

* Structure of actor declaration in the feature is the following:

```json
{
  "name": "actor-name",
  "kind": "stateless_actor",
  "dependency": "actor key in ioc"
}
```

* Actor naming guidelines:
    * Actor name in feature is representative of the action it performs on the entity
    * Actor name is written in English using `lisp-case`
    * Actor name shouldn't contain word `actor` in it
* Structure of chain declaration in the feature is the following:

```json
{
  "id": "chain-name",
  "externalAccess": false,
  "steps": [],
  "exceptional": []
}
```

* Chain declaration guidelines:
    * Chain name is representative of the action it's supposed to do
    * Chain name is written in English using `lisp-case`
    * Chain declaration must contain field `externalAccess` to clarify if this chain is external or not

#### Collections
* Collection naming guidelines:
    * Collection name represents entity it stores
    * Collection name written in English using `snake_case` style
    * Collection name should be singular i.e. no plurals allowed
* Collections should be created in the features where they're used

**Example**: 
```java
"raw_data"
"inactive_user_account"
```